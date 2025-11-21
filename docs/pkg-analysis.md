# pkg 模块详细分析

本文档详细分析 realtime-ai 项目中 `pkg` 目录下每个模块的用途和实现。

## 目录

- [1. Pipeline 模块](#1-pipeline-模块)
- [2. Connection 模块](#2-connection-模块)
- [3. Elements 模块](#3-elements-模块)
- [4. Trace 模块](#4-trace-模块)
- [5. ASR 模块](#5-asr-模块)
- [6. TTS 模块](#6-tts-模块)
- [7. Audio 模块](#7-audio-模块)
- [8. Tokenizer 模块](#8-tokenizer-模块)
- [9. Server 模块](#9-server-模块)
- [10. Utils 模块](#10-utils-模块)
- [11. Proto 模块](#11-proto-模块)
- [总结](#总结)

---

## 1. Pipeline 模块 (pkg/pipeline)

### 核心概念

Pipeline 模块是整个系统的核心，实现了一个**基于 GStreamer 思想的流式数据处理框架**，用于音视频和文本数据的流式传输。

### 关键组件

#### 1.1 数据结构 (pipeline.go:12-71)

**PipelineMessage** - 统一的管道消息封装:
- 支持 4 种消息类型: Audio(音频)、Video(视频)、Data(数据)、Command(命令)
- 包含 SessionID、Timestamp、元数据等通用字段
- 封装了 AudioData、VideoData、TextData 三种具体数据类型

**AudioData** (pipeline.go:12-19): 音频数据块
- 包含原始数据、采样率、声道数、媒体类型(如 audio/x-raw、audio/x-opus)、编解码器信息

**VideoData** (pipeline.go:21-31): 视频数据块
- 包含原始数据、宽高、媒体类型、格式、帧率、编解码器信息

**TextData** (pipeline.go:33-37): 文本数据块
- 包含原始数据、文本类型、时间戳

#### 1.2 Pipeline 核心实现 (pipeline.go:73-186)

**Pipeline 结构**:
```go
type Pipeline struct {
    name     string      // 管道名称
    bus      Bus         // 事件总线
    elements []Element   // 元素列表(处理单元)
}
```

**核心功能**:
- `AddElement/AddElements` (pipeline.go:89-103): 添加处理元素，自动设置事件总线
- `Link(a, b)` (pipeline.go:107-138): 连接两个元素，创建数据流通道，返回取消函数用于断开
- `Push` (pipeline.go:144-153): 向第一个元素输入数据
- `Pull` (pipeline.go:156-161): 从最后一个元素输出数据
- `Start/Stop` (pipeline.go:163-185): 启动/停止所有元素和事件总线

#### 1.3 Element 接口 (element.go:18-34)

**Element 接口** - 处理单元的抽象:
```go
type Element interface {
    Init(ctx context.Context) error
    In() chan<- *PipelineMessage    // 输入通道
    Out() <-chan *PipelineMessage   // 输出通道
    Start/Stop() error              // 生命周期管理
    SetBus(bus Bus)                 // 设置事件总线
    SetProperty/GetProperty         // 属性管理
}
```

**BaseElement** (element.go:36-128) - 基础实现:
- 提供双向通道 InChan 和 OutChan
- 实现属性系统(PropertyDesc 描述，支持类型检查、读写控制)
- 子类可继承并实现具体的处理逻辑

#### 1.4 EventBus 事件总线 (bus.go:52-182)

**用途**: 元素间的异步事件通信，解耦数据流和控制流

**EventType 支持** (bus.go:13-25):
- Error/Warning: 错误和警告
- PartialResult/FinalResult: 部分/最终结果
- BargeIn/Interrupted: 打断相关
- Started/Stopped: 状态变化
- VADSpeechStart/VADSpeechEnd: 语音活动检测

**实现机制**:
- 订阅-发布模式(Subscribe/Publish)
- 支持异步后台处理(通过 eventChan 缓冲队列)
- 非阻塞投递(通道满时丢弃事件并警告)

#### 1.5 ClearableChan 可清空通道 (chan.go:8-58)

**特性**: 
- 带互斥锁保护的通道封装
- 支持清空操作(Clear)，用于打断场景(如用户插话时清空待播放音频)
- 非阻塞发送(Send)，通道满时打印日志

---

## 2. Connection 模块 (pkg/connection)

### 核心接口

#### 2.1 RTCConnection 接口 (connection.go:34-52)

统一的连接抽象，支持多种传输协议:
```go
type RTCConnection interface {
    PeerID() string                              // 连接唯一标识
    RegisterEventHandler(handler ConnectionEventHandler) // 注册事件处理器
    SendMessage(msg *PipelineMessage)            // 发送消息
    Close() error                                // 关闭连接
}
```

#### 2.2 ConnectionEventHandler 接口 (connection.go:8-32)

定义连接事件回调:
- `OnConnectionStateChange`: 连接状态变化(使用 WebRTC 状态枚举)
- `OnMessage`: 收到消息回调
- `OnError`: 错误回调

### 四种连接实现

#### 2.3 WebSocket 连接 (ws_connection.go)

**用途**: 基于 WebSocket 的简单连接，适合 Web 客户端

**实现细节**:
- 使用 Gorilla WebSocket 库
- 双 goroutine 模式: readPump(读取) + writePump(写入)
- WSMessage 协议: JSON 格式，包含 type 和 payload
- 支持音频(audio)和文本(text)两种消息类型
- 非阻塞发送(通道满时丢弃消息)

**消息处理**:
- 接收: WebSocket → JSON 解析 → PipelineMessage → OnMessage 回调
- 发送: PipelineMessage → JSON 封装 → WebSocket

#### 2.4 WebRTC 连接 (rtc_connection.go)

**用途**: 基于 Pion WebRTC 的实时音视频连接，支持 P2P 和低延迟

**核心组件**:
- **PeerConnection**: Pion WebRTC 核心对象
- **音频编解码**: 使用 Opus 编解码器
  - 默认参数: 48kHz, 单声道, 50kbps
  - 设置: VoIP 模式, 复杂度 10, DTX 开启
- **TrackLocalStaticSample**: 本地音频轨道(发送)
- **TrackRemote**: 远端音频轨道(接收)
- **DataChannel**: 用于文本数据传输

**音频处理流程**:
1. **发送端** (rtc_connection.go:229-273): 
   - PCM 数据 → Int16 转换 → Opus 编码 → WriteSample 到本地轨道
   - 支持重采样(当前已注释)

2. **接收端** (rtc_connection.go:284-332):
   - ReadRTP → Opus 解码 → Int16 转 PCM → PipelineMessage → OnMessage 回调
   - 在独立 goroutine 中循环读取

**DataChannel** (rtc_connection.go:334-351):
- OnDataChannel 回调处理
- 文本消息直接发送原始字节

#### 2.5 gRPC 连接 (grpc_connection.go)

**用途**: 基于 gRPC 双向流的连接，适合服务端到服务端通信

**实现特点**:
- 使用 Protocol Buffers 定义的 StreamingAIService
- 双向流: `BiDirectionalStreamingServer`
- 完整的 PipelineMessage ↔ protobuf 转换

**消息类型映射** (grpc_connection.go:134-189):
- Audio: AudioFrame (包含采样率、声道、编解码器等)
- Video: VideoFrame (包含分辨率、帧率等)
- Text: TextMessage
- Control: 控制消息(状态变化、错误等)

**生命周期管理**:
- context.WithCancel 控制
- WaitGroup 等待 goroutine 退出
- receiveLoop 独立 goroutine 接收消息

**状态同步** (grpc_connection.go:262-275):
- 支持通过 Control 消息同步连接状态
- 映射 protobuf 状态到 WebRTC 状态枚举

#### 2.6 本地连接 (local_connection.go)

**用途**: 直接使用本地音频设备(麦克风+扬声器)，用于本地测试

**核心依赖**: malgo 库(跨平台音频 I/O)

**音频设备配置** (local_connection.go:16-22):
- **采集设备**: 16kHz, 单声道, 20ms 周期
- **播放设备**: 48kHz, 单声道, 20ms 周期
- 格式: S16 (16-bit PCM)

**采集实现** (local_connection.go:111-152):
- Data 回调中拷贝音频数据
- 封装成 PipelineMessage 后调用 OnMessage

**播放实现** (local_connection.go:154-266):
- 100ms 预缓冲机制(防止爆音)
- 双缓冲: audioBuffer(缓冲阶段) + playAudioBuffer(播放阶段)
- 独立 goroutine 接收待播放消息
- Data 回调中从 playAudioBuffer 拷贝数据到输出

**调试支持**:
- 环境变量 `DUMP_LOCAL_PLAYBACK=true` 开启播放数据导出
- 使用 audio.Dumper 保存播放的音频数据

---

## 3. Elements 模块 (pkg/elements)

Elements 是 Pipeline 的处理单元，每个 Element 实现特定功能。所有 Element 继承自 `BaseElement`，通过 In/Out 通道连接。

### 3.1 音频编解码元素

#### AudioResampleElement (audio_resample_element.go)

**用途**: 音频重采样，转换采样率和声道数

**实现**:
- 使用 FFmpeg (astiav) 的重采样器
- 支持单声道↔立体声转换
- 只处理 `audio/x-raw` 类型
- 输入参数: 输入采样率/声道、输出采样率/声道

**应用场景**: 
- 连接不同采样率的设备(如 16kHz 采集 → 48kHz 播放)
- 适配不同 AI 服务的音频要求

#### OpusEncodeElement (opus_encode_element.go)

**用途**: PCM → Opus 编码

**实现**:
- 使用 hraban/opus 库
- 默认参数: 64kbps, 复杂度10(最高质量)
- 输入: `audio/x-raw` (PCM)
- 输出: `audio/x-opus`
- 最大帧大小: 1275 字节

#### OpusDecodeElement (opus_decode_element.go)

**用途**: Opus → PCM 解码

**实现**:
- Opus 解码为 PCM
- 支持 dump 调试(环境变量 `DUMP_OPUS_DECODED=true`)
- 输出: `audio/x-raw`
- 解码缓冲: 1920 samples (立体声 * 960)

### 3.2 语音识别元素

#### WhisperSTTElement (whisper_stt_element.go)

**用途**: 使用 OpenAI Whisper API 进行语音转文字

**核心特性**:
- **VAD 集成**: 
  - `VADEnabled=true`: 监听 `EventVADSpeechStart/End`，仅在说话时识别
  - `VADEnabled=false`: 持续识别
- **流式识别**: 使用 `asr.StreamingRecognizer`
- **部分结果**: 支持 `EnablePartialResults`
- **音频缓冲**: 10 秒缓冲(16kHz × 2 bytes × 10s)

**工作流程**:
1. 订阅 VAD 事件(如果启用)
2. processAudio (whisper_stt_element.go:290-336): 接收音频并缓冲
3. handleVADEvents (whisper_stt_element.go:338-375): VAD 结束时触发识别
4. handleResults (whisper_stt_element.go:410-483): 处理识别结果，发布事件和输出消息

**属性系统**: 支持运行时设置 language、model、enable_partial_results、vad_enabled

### 3.3 语音合成元素

#### AzureTTSElement (azure_tts_element.go)

**用途**: 使用 Azure Speech Service 进行文本转语音

**实现**:
- 使用 REST API (HTTP POST)
- SSML 格式输入
- 默认: zh-CN, XiaoxiaoNeural 声音, 24kHz 单声道 PCM
- 环境变量: `AZURE_SPEECH_KEY`, `AZURE_SPEECH_REGION`

**流程** (azure_tts_element.go:92-147):
1. 接收 `MsgTypeData` (文本)
2. 构建 SSML
3. 调用 Azure TTS API
4. 输出 `MsgTypeAudio` (PCM 数据)

### 3.4 VAD 元素

#### SileroVADElement (vad_element.go)

**用途**: 语音活动检测 (Voice Activity Detection)

**实现**:
- 使用 Silero VAD 模型 (streamer45/silero-vad-go)
- 仅支持 16kHz 采样率
- 两种模式 (vad_element.go:18-26):
  - **Passthrough**: 透传所有音频，仅发布事件
  - **Filter**: 仅在检测到语音时透传音频

**配置参数** (vad_element.go:36-42):
- `Threshold`: 检测阈值 (0-1, 默认 0.5)
- `MinSilenceDurMs`: 最小静音持续时间 (默认 100ms)
- `SpeechPadMs`: 语音前后填充时间 (默认 30ms)
- `ModelPath`: 模型文件路径

**工作流程** (vad_element.go:198-287):
1. 将字节数据转 int16 samples
2. 以 512 samples (32ms @ 16kHz) 为单位检测
3. 检测到语音开始/结束时发布 `EventVADSpeechStart/End`
4. 根据模式决定是否透传音频

### 3.5 播放缓冲元素

#### PlayoutSinkElement (playout_sink_element.go)

**用途**: 管理播放缓冲，实现平滑播放

**核心功能**:
- 使用 `audio.PlayoutBuffer` 缓冲音频
- 双协程设计 (playout_sink_element.go:91-168):
  - 协程1: 接收音频写入缓冲
  - 协程2: 定时(5ms)从缓冲读取并输出(20ms 一帧)
- 监听 `EventInterrupted` 事件(用于打断)
- 支持 dump 调试 (`DUMP_LOCAL_AUDIO=true`)

**应用**: 连接到 Connection 的播放端，确保音频平滑输出

### 3.6 LLM 元素

#### OpenAIRealtimeAPIElement (openai_realtimeapi_element.go)

**用途**: 集成 OpenAI Realtime API，实现端到端语音对话

**核心功能**:
- WebSocket 连接 OpenAI Realtime API
- 支持音频输入/输出、文本输入
- 自动语音转文字 (Input Audio Transcription)
- 服务端 VAD (Server-side VAD)
- 打断检测 (`InputAudioBufferSpeechStarted` → `EventInterrupted`)

**配置** (openai_realtimeapi_element.go:151-168):
```go
Modalities: [Text, Audio]
Voice: Shimmer
OutputAudioFormat: PCM16
TurnDetection: ServerVAD (threshold=0.7, silence=800ms)
```

**事件处理** (openai_realtimeapi_element.go:98-146):
- `ResponseAudioDelta`: 累积音频数据
- `ResponseAudioDone`: 输出完整音频帧
- `ResponseAudioTranscriptDone`: 打印 AI 回复文本
- `ConversationItemInputAudioTranscriptionCompleted`: 打印用户问题

#### TranslateElement (translate_element.go)

**用途**: 实时翻译文本

**支持提供商** (translate_element.go:22-29):
- OpenAI (gpt-4o-mini)
- Gemini (gemini-2.0-flash-exp)

**特性**:
- 流式翻译(Streaming): 降低延迟，发布 `EventPartialResult`
- 自动检测源语言 (`SourceLang="auto"`)
- 自定义翻译提示(SystemPrompt)

**工作流程** (translate_element.go:132-182):
1. 接收 `MsgTypeData` 文本
2. 调用 LLM API 翻译
3. 输出翻译后的文本(保留 TextType)
4. 发布 `EventFinalResult` 事件

---

## 4. Trace 模块 (pkg/trace)

### 用途

实现 OpenTelemetry 分布式追踪，用于性能监控和问题诊断。

### 核心功能 (trace.go)

**Config 配置** (trace.go:33-47):
- `ServiceName/Version`: 服务标识
- `Environment`: 环境(dev/staging/prod)
- `ExporterType`: 导出器类型
  - `stdout`: 控制台输出(调试)
  - `otlp`: OTLP gRPC 导出(生产环境)
  - `none`: 禁用追踪
- `OTLPEndpoint`: OTLP 接收端地址(如 localhost:4317)
- `SamplingRate`: 采样率(0.0-1.0)

**初始化流程** (trace.go:62-135):
1. 创建 Resource (包含服务信息和语义约定)
2. 创建 Exporter (根据配置选择)
3. 创建 TracerProvider (带采样器和批处理)
4. 设置全局 Propagator (支持分布式追踪上下文传播)

**辅助模块**:
- `attributes.go`: 预定义属性常量
- `ai.go`: AI 相关 span 创建(LLM 调用、STT、TTS 等)
- `pipeline.go`: Pipeline 元素追踪
- `connection.go`: 连接事件追踪

---

## 5. ASR 模块 (pkg/asr)

### 用途

提供统一的 ASR 接口，支持多种语音识别提供商。

### 核心接口 (interface.go)

**Provider 接口** (interface.go:94-116):
```go
type Provider interface {
    Name() string
    Recognize(ctx, audio, audioConfig, config) (*RecognitionResult, error)
    StreamingRecognize(ctx, audioConfig, config) (StreamingRecognizer, error)
    SupportsStreaming() bool
    SupportedLanguages() []string
    Close() error
}
```

**StreamingRecognizer 接口** (interface.go:80-92):
- `SendAudio`: 发送音频数据
- `Results()`: 返回结果通道
- `Close()`: 关闭识别器

**数据结构**:
- `RecognitionResult` (interface.go:12-34): 识别结果(文本、置信度、是否最终结果等)
- `AudioConfig` (interface.go:36-49): 音频格式(采样率、声道、编码、位深)
- `RecognitionConfig` (interface.go:51-78): 识别配置(语言、模型、部分结果、提示词等)

**实现**: whisper.go 实现了 OpenAI Whisper API 的 Provider

---

## 6. TTS 模块 (pkg/tts)

### 用途

提供统一的 TTS 接口，支持多种语音合成提供商。

### 核心接口 (provider.go)

**TTSProvider 接口** (provider.go:32-51):
```go
type TTSProvider interface {
    Name() string
    Synthesize(ctx, req) (*SynthesizeResponse, error)
    GetSupportedVoices() []string
    GetDefaultVoice() string
    ValidateConfig() error
}
```

**StreamingTTSProvider 接口** (provider.go:53-61) - 扩展:
- `StreamSynthesize`: 流式合成，返回音频块通道

**数据结构**:
- `SynthesizeRequest` (provider.go:16-21): 合成请求(文本、声音、语言、选项)
- `SynthesizeResponse` (provider.go:23-28): 合成响应(音频数据、格式、时长)
- `AudioFormat` (provider.go:8-13): 音频格式描述

**实现**: openai_provider.go 实现了 OpenAI TTS API

---

## 7. Audio 模块 (pkg/audio)

### 7.1 Resample (resample.go)

**用途**: FFmpeg 音频重采样

**实现**:
- 使用 astiav (FFmpeg Go 绑定)
- 支持采样率转换和声道布局转换
- 自动处理缓冲区对齐和填充

**核心方法** (resample.go:75-169):
- `NewResample`: 创建重采样器，配置输入/输出参数
- `Resample`: 执行重采样，处理帧分配和数据转换
- `Free`: 释放 FFmpeg 资源

### 7.2 PlayoutBuffer (playout.go)

**用途**: 播放缓冲区，实现平滑播放

**核心特性**:
- 24kHz → 48kHz 自动重采样
- 200ms 预缓冲(10帧 × 20ms)
- `accumulating` 模式: 积累足够数据后开始播放
- `ReadFrame` (playout.go:68-100): 固定 20ms 帧输出
- `Clear` (playout.go:103-109): 清空缓冲并重新积累(用于打断场景)

**常量定义** (playout.go:10-24):
```go
InputSampleRate  = 24000
OutputSampleRate = 48000
Channels = 1
BytesPerSample = 2
SamplesPerFrame24kHz = 480  // 20ms @ 24kHz
SamplesPerFrame48kHz = 960  // 20ms @ 48kHz
```

### 7.3 Dumper (dumper.go)

**用途**: 保存音频数据到 WAV 文件(调试工具)

**特性**:
- 流式 WAV 写入(WavStreamWriter)
- 每次写入后自动更新 WAV 头部 (dumper.go:47-64, 135-160)
- 文件名自动加时间戳和参数标记
- 线程安全(sync.Mutex)

**使用示例** (dumper.go:172-227):
```go
dumper, err := audio.NewDumper("tag_name", 48000, 1)
dumper.Write(audioData)
dumper.Close()
```

---

## 8. Tokenizer 模块 (pkg/tokenizer)

### 用途

提供句子分词接口，用于流式文本断句。

### SentenceTokenizer 接口 (tokenizer.go:12-33)

**核心方法**:
- `Feed(text)`: 批量输入文本，返回句子数组
- `FeedChar(char)`: 流式输入字符，返回完整句子
- `Reset()`: 重置分词器状态

**实现**: rule_boundary_tokenizer.go 实现了基于规则的断句器

**应用场景**: 
- 实时翻译时按句子分割
- TTS 合成时按句子处理

---

## 9. Server 模块 (pkg/server)

### RTCServer (server.go)

**用途**: WebRTC 信令服务器，处理 SDP 协商

**核心功能**:
- `Start()` (server.go:50-87): 初始化 WebRTC API 和 ICE UDP Mux
- `HandleNegotiate()` (server.go:90-167): HTTP 处理器，接收 Offer 返回 Answer

**配置** (server_config.go):
- `ICELite`: 启用 ICE Lite 模式
- `Endpoint`: NAT 1:1 IP 映射
- `RTCUDPPort`: UDP 端口

**工作流程**:
1. 接收客户端 SDP Offer
2. 创建 PeerConnection 和 RTCConnection
3. 调用 `onConnectionCreated` 回调
4. 生成 SDP Answer 并返回

**gRPC Server** (grpc_server.go):
- 实现 StreamingAIService gRPC 服务
- 支持双向流式音视频传输
- 与 RTCConnection 抽象无缝集成

---

## 10. Utils 模块 (pkg/utils)

### utils.go

**音频格式转换**:
- `Int16SliceToByteSlice` (utils.go:4-12): int16 PCM → 字节(小端序)
- `ByteSliceToInt16Slice` (utils.go:15-32): 字节 → int16 PCM(小端序)

**用途**: 在音频编解码和处理时转换数据格式

**实现细节**:
```go
// 小端序: 低位字节在前，高位字节在后
out[2*i] = byte(v)        // 低位
out[2*i+1] = byte(v >> 8) // 高位
```

---

## 11. Proto 模块 (pkg/proto)

### 用途

Protocol Buffers 定义，用于 gRPC 通信。

**文件结构**:
- `streamingai/v1/streaming_ai.proto`: 定义了双向流式 AI 服务
- `streaming_ai.pb.go`: 生成的 Go 代码
- `streaming_ai_grpc.pb.go`: 生成的 gRPC 服务代码

**核心消息类型**:
- `StreamMessage`: 统一消息封装
- `AudioFrame`: 音频帧(采样率、声道、编解码器)
- `VideoFrame`: 视频帧(分辨率、帧率、格式)
- `TextMessage`: 文本消息
- `ControlMessage`: 控制消息(状态变化、错误)

**服务定义**:
```protobuf
service StreamingAIService {
    rpc BiDirectionalStreaming(stream StreamMessage) returns (stream StreamMessage);
}
```

---

## 总结

整个 `pkg` 目录实现了一个**完整的实时 AI 对话系统框架**。

### 架构层次

```
┌─────────────────────────────────────────────────────────┐
│                     应用层                                │
│            (cmd/server, examples)                        │
└─────────────────────────────────────────────────────────┘
                            │
┌─────────────────────────────────────────────────────────┐
│                    服务层                                 │
│   server (WebRTC/gRPC 信令)  │  trace (监控)            │
└─────────────────────────────────────────────────────────┘
                            │
┌─────────────────────────────────────────────────────────┐
│                    处理层                                 │
│  elements (VAD/STT/TTS/LLM/编解码/重采样)                │
└─────────────────────────────────────────────────────────┘
                            │
┌─────────────────────────────────────────────────────────┐
│                    管道层                                 │
│  pipeline (数据流框架 + 事件总线)                         │
└─────────────────────────────────────────────────────────┘
                            │
┌─────────────────────────────────────────────────────────┐
│                    传输层                                 │
│  connection (WebRTC/WebSocket/gRPC/Local)                │
└─────────────────────────────────────────────────────────┘
                            │
┌─────────────────────────────────────────────────────────┐
│                  抽象/工具层                              │
│  asr/tts (Provider抽象) │ audio/tokenizer/utils (工具)  │
└─────────────────────────────────────────────────────────┘
```

### 设计特点

1. **松耦合**: 通过接口抽象(Provider、Element、Connection)实现可扩展性
2. **流式处理**: 全链路支持流式数据(音频流、识别流、合成流)
3. **事件驱动**: EventBus 解耦控制流和数据流
4. **多协议支持**: 同时支持 WebRTC、WebSocket、gRPC、本地设备
5. **可观测性**: 完整的 OpenTelemetry 追踪支持
6. **调试友好**: Dumper、环境变量控制的日志和数据导出

### 典型数据流

```
[麦克风/WebRTC] 
  ↓
AudioResampleElement (16kHz)
  ↓
VADElement (语音检测)
  ↓ EventVADSpeechStart/End
  ↓
WhisperSTTElement (语音识别)
  ↓ text
  ↓
TranslateElement (可选翻译)
  ↓ translated text
  ↓
OpenAIRealtimeAPIElement / LLM (AI 处理)
  ↓ response text
  ↓
AzureTTSElement (语音合成)
  ↓ audio 24kHz
  ↓
AudioResampleElement (48kHz)
  ↓
PlayoutSinkElement (播放缓冲)
  ↓ audio frames (20ms)
  ↓
[扬声器/WebRTC]
```

### 使用场景

这个架构非常适合构建:
- 多语言实时语音对话系统
- 语音翻译应用
- 语音助手/智能客服
- 远程会议翻译
- 实时字幕生成
- 语音驱动的 AI Agent

### 扩展性

添加新功能只需:
1. **新的 AI 服务**: 实现 `asr.Provider` 或 `tts.TTSProvider` 接口
2. **新的传输协议**: 实现 `connection.RTCConnection` 接口
3. **新的处理单元**: 继承 `pipeline.BaseElement` 实现 `Element` 接口
4. **新的事件类型**: 在 `pipeline/bus.go` 中添加 `EventType` 常量

### 性能优化

- **并发处理**: 所有 Element 独立 goroutine 运行
- **缓冲管理**: 通道缓冲、PlayoutBuffer 预缓冲
- **非阻塞设计**: 通道满时丢弃而非阻塞
- **资源复用**: 编解码器、重采样器等重复使用
- **批处理**: Trace 使用 Batcher 减少网络开销

### 环境变量配置

调试时可用的环境变量:
- `DUMP_LOCAL_PLAYBACK=true`: 导出本地播放音频
- `DUMP_LOCAL_AUDIO=true`: 导出 PlayoutSink 音频
- `DUMP_OPUS_DECODED=true`: 导出 Opus 解码后音频
- `DUMP_OPENAI_AUDIO=true`: 导出 OpenAI 响应音频
- `TRACE_EXPORTER=stdout|otlp|none`: 追踪导出器
- `OTEL_EXPORTER_OTLP_ENDPOINT`: OTLP 端点地址
- `ENVIRONMENT=dev|staging|prod`: 运行环境
- `AZURE_SPEECH_KEY`: Azure Speech 密钥
- `AZURE_SPEECH_REGION`: Azure Speech 区域
- `OPENAI_API_KEY`: OpenAI API 密钥
