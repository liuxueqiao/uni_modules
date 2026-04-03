# LXQ-NativeRecorderModule

基于 UTS 开发的原生录音流式插件，能够实时返回音频 PCM 帧数据（Base64），并 **支持 App 切入后台或手机息屏后依旧坚挺录音**。

## 平台支持及特性
- **Android**: 
  - 基于底层的 `android.media.AudioRecord`，在子线程实时读取 `PCM 16BIT` 音频流并进行 Base64 编码抛出。
  - **后台保活**: 启动前台服务 (`Service`) 并申请 `WakeLock` 唤醒锁，强制在系统通知栏驻留“正在后台录音”通知，避免进程在息屏后被系统休眠杀死。
- **iOS**: 
  - 基于核心音频引擎 `AVAudioEngine`，使用 `installTap` 在音频输入节点上实时监听 `AVAudioPCMBuffer` 数据并转码抛出。
  - **后台保活**: 设置音频会话类别为 `PlayAndRecord`，并在 `info.plist` 中声明了 `UIBackgroundModes: audio`，确保退到后台也能持续采集。
- **HarmonyOS**: 
  - 基于原生 `AudioCapturer` 获取 `ArrayBuffer` 的音频帧数据，并使用 `@ohos.util` 内置工具进行 Base64 编码抛出。
  - **后台保活**: 结合 `@ohos.resourceschedule.backgroundTaskManager`，开启 `AUDIO_RECORDING` 长时任务模式保活。

## 核心优势对比普通录音插件
1. **实时帧流返回**：录音时不会将内容写入文件，而是实时抓取底层录音硬件的 PCM 裸音频流，主要用于对接**实时语音识别(ASR)大模型**、**WebSocket 语音对讲**等场景。
2. **完全后台运行**：无论手机处于什么状态（退到后台、锁屏亮屏），这套机制都会强力运行直到您主动调用 `stopRecord` 销毁。

## 快速使用

```typescript
import { startRecord, stopRecord } from '@/uni_modules/LXQ-NativeRecorderModule'

// 开始录制
startRecord({
  // 采样率，默认 16000
  sampleRate: 16000,
  // 通道数，1 单声道，2 双声道
  channelCount: 1,
  
  // 状态与异常的回调
  onEvent: (event) => {
    console.log("录制事件：", event)
    if(event.code === 0) {
       console.log("已开始后台保活录音...")
    } else if(event.code === 1) {
       console.log("已正常结束...")
    } else if(event.code === -1) {
       console.error("发生错误：", event.msg)
    }
  },
  
  // 实时音频帧回调（频率极高，请勿在此处做重计算）
  onFrameRecorded: (frame) => {
    // 获取到 Base64 编码的音频 PCM 帧数据
    // App 切入后台或锁屏后，这里依旧会不断执行！
    // console.log("收到音频帧：", frame.data)
  }
})

// 结束录制，会同时销毁系统的后台保活服务和所有锁
setTimeout(() => {
    stopRecord()
}, 30000)
```

## 配置项及接口定义

参考 `interface.uts` 中的定义：
* `startRecord(options: StartRecordOptions)` - 启动保活录音
* `stopRecord()` - 停止录音并结束保活服务

```typescript
export type StartRecordOptions = {
    sampleRate?: number;
    channelCount?: number;
    format?: string;
    onEvent?: (res: RecordEventParams) => void;
    onFrameRecorded?: (res: RecordFrameParams) => void;
}
```

## 注意事项
1. **权限申请**: 插件内部已配置清单所需的权限（如麦克风、唤醒锁、前台服务等），但在前端调用前，建议您先主动检查并请求“麦克风”权限，以防部分设备拦截。
2. **性能损耗**: 由于是高频 Base64 字符串回传（约每 128ms 一次回传），以及开启了唤醒锁/长时任务，录音期间会增加一定的耗电量，不用时请**务必**主动调用 `stopRecord`，否则该服务会一直运行至 App 被强杀。