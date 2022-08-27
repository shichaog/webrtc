ADM

audio device module，采集和播放声音都需要具体的硬件支持，Linux、Android、iOS、windows、mac都有不同硬件驱动和系统API可供调用，为了屏蔽不同操作系统提供的API差异，类似于ACM模块给编码器提供统一的接口、APM模块给音频处理算法提供统一的接口一样，ADM模块也提供了`AudioDeviceModule`接口类。设备模块接口类需要提供设备枚举、设备选择、播放采集、混音、音量控制等功能。

描述设备的接口类实现定义于AudioDeviceModuleImpl，这里使用到了关键成员变量是

  AudioDeviceBuffer **audio_device_**buffer_;

  std::unique_ptr<AudioDeviceGeneric> **audio_device_**;

，其中audio_device_是和平台相关的，而**audio_device_**buffer_则维护了音频设备的缓冲区，而audio_device_则代表了平台硬件设备，采集、播放、音量控制等都通过这个对象的接口实现。即AudioDeviceModuleImpl对象调用相应的采集、播放、音量控制等方法时，就会调用audio_device_对象里的方法完成响应的动作。

```c++
class AudioDeviceModuleImpl:public AudioDeviceModule{
   public:
  enum PlatformType {
    kPlatformNotSupported = 0,
    kPlatformWin32 = 1,
    kPlatformWinCe = 2,
    kPlatformLinux = 3,
    kPlatformMac = 4,
    kPlatformAndroid = 5,
    kPlatformIOS = 6
  };
    // Retrieve the currently utilized audio layer
  int32_t ActiveAudioLayer(AudioLayer* audioLayer) const override;

  // Full-duplex transportation of PCM audio
  int32_t RegisterAudioCallback(AudioTransport* audioCallback) override;
    // Device enumeration
  int16_t PlayoutDevices() override;
  int16_t RecordingDevices() override;
  int32_t PlayoutDeviceName(uint16_t index,
                            char name[kAdmMaxDeviceNameSize],
                            char guid[kAdmMaxGuidSize]) override;
  int32_t RecordingDeviceName(uint16_t index,
                              char name[kAdmMaxDeviceNameSize],
                              char guid[kAdmMaxGuidSize]) override;
  
    // Device selection
  int32_t SetPlayoutDevice(uint16_t index) override;
  int32_t SetPlayoutDevice(WindowsDeviceType device) override;
  int32_t SetRecordingDevice(uint16_t index) override;
  int32_t SetRecordingDevice(WindowsDeviceType device) override;
  
    // Audio transport initialization
  int32_t PlayoutIsAvailable(bool* available) override;
  int32_t InitPlayout() override;
  bool PlayoutIsInitialized() const override;
  int32_t RecordingIsAvailable(bool* available) override;
  int32_t InitRecording() override;
  bool RecordingIsInitialized() const override;

  // Audio transport control
  int32_t StartPlayout() override;
  int32_t StopPlayout() override;
  bool Playing() const override;
  int32_t StartRecording() override;
  int32_t StopRecording() override;
  bool Recording() const override;
  ...
     private:
  PlatformType Platform() const;
  AudioLayer PlatformAudioLayer() const;

  AudioLayer audio_layer_;
  PlatformType platform_type_ = kPlatformNotSupported;
  bool initialized_ = false;
#if defined(WEBRTC_ANDROID)
  // Should be declared first to ensure that it outlives other resources.
  std::unique_ptr<AudioManager> audio_manager_android_;
#endif
  AudioDeviceBuffer audio_device_buffer_;
  std::unique_ptr<AudioDeviceGeneric> audio_device_;
}
```



AudioDeviceGeneric这个类描述了设备的通用能力，属于接口类，不同平台会继承这个类，通过多态的方式访问对应平台的方法，因而在AudioDeviceModuleImpl可以使用一个接口而不需要关心不同操作系统提供的不同方法。

```c++
int32_t AudioDeviceModuleImpl::CreatePlatformSpecificObjects() {

// Dummy ADM implementations
  audio_device_.reset(new AudioDeviceDummy());
// Dummy File ADM implementations
  audio_device_.reset(FileAudioDeviceFactory::CreateFileAudioDevice());
// Windows ADM implementation.
  audio_device_.reset(new AudioDeviceWindowsCore()
                      
// Java audio for input and AAudio for output audio (i.e. mixed APIs).
  audio_device_.reset(new AudioDeviceTemplate<AudioRecordJni, AAudioPlayer>(
        kAndroidAAudioAudio, audio_manager));
// Linux ADM implementation.
   audio_device_.reset(new AudioDeviceLinuxALSA());
// iOS ADM implementation.
    audio_device_.reset(
        new ios_adm::AudioDeviceIOS(/*bypass_voice_processing=*/false));
// Mac OS X ADM implementation.
   audio_device_.reset(new AudioDeviceMac());
```



其在webrtc_voice_engine.h中被使用到

```c++
  // The audio device module.
  rtc::scoped_refptr<webrtc::AudioDeviceModule> adm_;
  rtc::scoped_refptr<webrtc::AudioEncoderFactory> encoder_factory_;
  rtc::scoped_refptr<webrtc::AudioDecoderFactory> decoder_factory_;
  rtc::scoped_refptr<webrtc::AudioMixer> audio_mixer_;
  // The audio processing module.
```



这样就把ADM、APM关联起来了。