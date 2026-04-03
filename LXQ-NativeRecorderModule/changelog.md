## 1.0.1
- 重构：对齐 `UniPlugin-Hello-AS/NativeRecorderModule`，将录音模式从“文件写入”改为“实时 PCM 帧数据流 Base64 返回”
- 新增：全面支持后台录音与息屏保活
  - Android：新增前台 Service 服务与 `WakeLock` 唤醒锁保活
  - iOS：升级为 `AVAudioEngine` 底层音频采集，增加 `UIBackgroundModes(audio)` 声明
  - HarmonyOS：采用 `AudioCapturer` 实时流，结合 `backgroundTaskManager(AUDIO_RECORDING)` 开启系统级长时任务保活