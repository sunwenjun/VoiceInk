# 音频电平计量与可视化实现分析（VoiceInk）

> 目标：全面拆解项目里“音频电平计量 + UI 可视化”的实际实现路径，并给一个可复用的完整案例，最后用 ANSI Art 时序图串起全流程。

## 1. 先说结论（TL;DR）

VoiceInk 的音频电平链路分成 4 层：

1. **采集层（CoreAudio 回调线程）**：`CoreAudioRecorder` 在 `AudioUnitRender` 回调里拿到 Float32 PCM 后，实时计算 **RMS/Peak** 并转 dBFS，写入线程安全的 `_averagePower/_peakPower`。  
2. **平滑层（Recorder 的高频定时器线程）**：`Recorder` 每 17ms 读取 dB 值，做 **[-60, 0] dB 归一化 + EMA 平滑**，产出 `[0,1]` 的 `AudioMeter`。  
3. **状态层（MainActor）**：将 `audioMeter` 发布给 SwiftUI，同时用阈值 `>0.01` 标记“本次会话是否检测到声音”，驱动无声提醒逻辑。  
4. **展示层（SwiftUI）**：`AudioVisualizer` 用 `TimelineView`（约 60Hz）做 15 柱动态波形，柱高由 `averagePower` + 正弦相位 + 中心增强共同决定。

这套实现兼顾了：
- 回调线程实时性（无 UI 操作、少分配）
- UI 动效稳定性（双重节奏：17ms 采样 + 16ms 动画）
- 并发安全（`NSLock` 保护 meter 与平滑状态）

---

## 2. 代码结构与职责映射

### 2.1 录音入口与状态切换

- 在 `WhisperState.toggleRecord` 的启动分支里，先构造输出 WAV 路径，再调用 `recorder.startRecording(...)`，成功后把 `recordingState` 设为 `.recording`。  
- 停止分支则会调用 `recorder.stopRecording()`，状态回收。  

这意味着：**音频可视化是否动起来，根状态由 `recordingState` 决定**。当状态变成 `.recording` 后，UI 才展示动态柱状图而不是静态占位。  

相关文件：`VoiceInk/Whisper/WhisperState.swift`。

### 2.2 计量数据来源：CoreAudioRecorder

`CoreAudioRecorder` 有两类核心数据：

- **音频数据路径**：`handleInputBuffer` → `AudioUnitRender` → `convertAndWriteToFile`
- **计量路径**：`handleInputBuffer` → `calculateMeters`

`calculateMeters` 的算法非常标准：

1. 遍历本次回调拿到的全部采样（含多声道）。
2. `sum += sample^2` 得到能量累计。
3. `peak = max(abs(sample))` 得到峰值幅度。
4. `rms = sqrt(sum / totalSamples)`。
5. 转 dBFS：
   - `avgDb = 20*log10(max(rms, 1e-6))`
   - `peakDb = 20*log10(max(peak, 1e-6))`
6. 用 `meterLock` 写入 `_averagePower/_peakPower`。

几个关键点：

- 使用 `max(..., 1e-6)` 避免 `log10(0)`。
- dB 结果本质是负值区间（安静接近 -∞，代码里初始与重置是 -160）。
- 该层只负责“**真实测量**”，不做 UI 友好处理（不归一化、不平滑）。

相关文件：`VoiceInk/CoreAudioRecorder.swift`。

### 2.3 UI 友好化：Recorder 的 17ms 定时器

`Recorder.startRecording` 在启动成功后会 `startAudioMeterTimer()`：

- 创建 `DispatchSourceTimer`
- 触发周期 17ms（约 58.8Hz）
- 在后台队列调用 `updateAudioMeter()`

`updateAudioMeter()` 的步骤：

1. 读取 `recorder.averagePower/peakPower`（线程安全 getter）。
2. 以 `[-60, 0] dB` 做线性归一化：
   - `< -60` 映射 0
   - `>= 0` 映射 1
   - 中间线性插值
3. 做 EMA（指数滑动平均）：
   - `smoothed = smoothed*0.6 + current*0.4`
4. 切回主线程发布 `@Published audioMeter`。
5. 若 `averagePower > 0.01`，标记检测到声音，避免后续“无声提醒”。

这里的设计核心是：**音频线程给原始真值，UI线程消费平滑值**，避免柱状图抖动和视觉噪声。

相关文件：`VoiceInk/Recorder.swift`。

### 2.4 可视化组件：AudioVisualizer

`RecorderStatusDisplay` 会根据状态切换：

- `.recording` → `AudioVisualizer`
- `.transcribing/.enhancing` → 文本+进度动画
- 其他 → `StaticVisualizer`

`AudioVisualizer` 的绘制模型：

- 15 根窄柱（固定间距）
- `TimelineView(.animation(minimumInterval: 0.016))` 按约 60Hz 重绘
- 每根柱高度：
  1. 取 `audioMeter.averagePower`（已是 0~1）
  2. `pow(amplitude, 0.7)` 提升低电平可见性
  3. 乘 `sin(time*8 + phase[i])` 的波动因子
  4. 乘中心增强（中间柱更高）
  5. 映射到 `[minHeight, maxHeight]`

所以它不是“逐柱频谱”，而是“**以整体音量驱动的动态波形感可视化**”。优点是轻量、丝滑、风格统一。

相关文件：
- `VoiceInk/Views/Recorder/RecorderComponents.swift`
- `VoiceInk/Views/Recorder/AudioVisualizerView.swift`
- `VoiceInk/Views/Recorder/MiniRecorderView.swift`

---

## 3. 完整案例（带数字推演）

下面给一个“你说一句话”的简化案例，演示单次计量如何变成 UI 柱高。

### 场景假设

- 设备采样回调里某帧窗口（简化）测得：
  - `rms = 0.1`
  - `peak = 0.25`

### Step A：dBFS 计算（CoreAudioRecorder）

- `avgDb = 20*log10(0.1) = -20 dB`
- `peakDb = 20*log10(0.25) ≈ -12.04 dB`

写入 `_averagePower=-20`, `_peakPower=-12.04`。

### Step B：归一化到 0~1（Recorder）

可视区间 `[-60, 0]`：

- `normalizedAverage = (-20 - (-60)) / 60 = 40/60 = 0.6667`
- `normalizedPeak = (-12.04 - (-60)) / 60 ≈ 0.7993`

### Step C：EMA 平滑

假设上一帧平滑值 `smoothedAverage=0.50`, `smoothedPeak=0.70`：

- `newAvg = 0.50*0.6 + 0.6667*0.4 = 0.5667`
- `newPeak = 0.70*0.6 + 0.7993*0.4 = 0.7397`

发布 `AudioMeter(averagePower:0.5667, peakPower:0.7397)`。

### Step D：可视化柱高（AudioVisualizer）

以某根柱某时刻：

- `amplitude = 0.5667`
- `boosted = pow(0.5667, 0.7) ≈ 0.671`
- 若该柱波动因子 `wave=0.8`
- 若中心增强 `centerBoost=0.9`
- `height = 4 + (0.671*0.8*0.9)*(28-4)`
- `height ≈ 4 + 0.483*24 ≈ 15.6`

于是你看到这根柱大约 16pt 高，且随着时间相位变化上下摆动。

---

## 4. 为什么这套实现合理

1. **线程分离明确**：计量在音频回调，UI在主线程，减少互相干扰。  
2. **视觉稳定**：EMA 0.6/0.4 不会过慢，也能过滤瞬时毛刺。  
3. **可视范围贴近语音**：-60dB 以下直接视作 0，背景噪声不至于“乱跳”。  
4. **动画和采样频率接近**：17ms 与 16ms 非常接近，画面连贯。

---

## 5. 可改进点（推理建议）

1. **峰值保持线（Peak Hold）**：当前 UI 只主要用 `averagePower`，可叠加一条短暂保持的 peak 指示，反馈更“专业”。
2. **自适应噪声地板**：固定 -60dB 在不同麦克风上可能偏硬，可动态估计噪声底。
3. **分层可视化模式**：当前是“整体响度波形感”，如需工程级监测可新增真实 VU/LUFS 面板。
4. **参数可配置**：把 `minVisibleDb`、EMA 系数、barCount 暴露到设置便于调优。

---

## 6. ANSI Art 时序图（端到端流程）

```text
+--------------------+      +----------------------+      +------------------+      +-------------------------+
| WhisperState       |      | Recorder             |      | CoreAudioRecorder|      | SwiftUI AudioVisualizer |
+--------------------+      +----------------------+      +------------------+      +-------------------------+
        |                              |                            |                               |
        | toggleRecord(start)          |                            |                               |
        |----------------------------->|                            |                               |
        |                              | startRecording(url)        |                               |
        |                              |--------------------------->| create AUHAL / start unit     |
        |                              |                            |------------------------------>|
        |                              | startAudioMeterTimer(17ms) |                               |
        |                              |==== periodic tick ========>|                               |
        |                              |                            | AudioUnitRender callback      |
        |                              |                            | calculateMeters(rms/peak->dB) |
        |                              |<===========================| averagePower / peakPower      |
        |                              | normalize[-60,0], EMA      |                               |
        |                              | publish @MainActor audioMeter                               |
        |                              |------------------------------------------------------------->|
        |                              |                            |        TimelineView(16ms)      |
        |                              |                            |<-------------------------------|
        |                              |                            |  calculateHeight(sin+boost)    |
        |                              |                            |------------------------------->|
        |                              |                            |       draw 15 bars              |
        |                              |                            |                               |
        | toggleRecord(stop)           |                            |                               |
        |----------------------------->| stopRecording()            |                               |
        |                              |--------------------------->| stop AU / reset meters         |
        |                              | set audioMeter=0           |                               |
        |                              |------------------------------------------------------------->|
        |                              |                            |                               |
```

---

## 7. 快速定位代码清单

- 录音状态启动/切换：`VoiceInk/Whisper/WhisperState.swift`
- dB 计量核心：`VoiceInk/CoreAudioRecorder.swift`
- 归一化与平滑：`VoiceInk/Recorder.swift`
- 状态到 UI 分发：`VoiceInk/Views/Recorder/RecorderComponents.swift`
- 动态波形绘制：`VoiceInk/Views/Recorder/AudioVisualizerView.swift`
- 容器视图承载：`VoiceInk/Views/Recorder/MiniRecorderView.swift`

