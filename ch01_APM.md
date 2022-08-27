# APM

APM(Audio Processing Module)提供了音频处理模块的集合，这里音频处理算法针对的是实时通信场景。APM逐帧对两路音频帧处理，其中一路的音频（near-end，采集信号）将会调用所有音频算法进行处理，这通过调用`ProcessStream()`实现，而另一路音（far-end,接收到的信号）频调用`ProcessReverseStream()`处理。

APM模块只接受10ms帧长的PCM数据，其帧长可以通过API `AudioProcessing::GetFrameSize()`获取，对于int16格式的输入API数据是交叉存放的，而浮点格式的音频输入则采用非交叉存放。

本篇不会深入算法原理以及相关实现的技巧，对此感兴趣读者可以参考《实时语音处理实践指南》一书。

APM模块使用的例子如下：

```c++
AudioProcessing* apm = AudioProcessingBuilder().Create();

AudioProcessing::Config config;
config.echo_canceller.enabled = true;
config.echo_canceller.mobile_mode = false;

config.gain_controller1.enabled = true;
config.gain_controller1.mode =
AudioProcessing::Config::GainController1::kAdaptiveAnalog;
config.gain_controller1.analog_level_minimum = 0;
config.gain_controller1.analog_level_maximum = 255;

config.gain_controller2.enabled = true;
config.high_pass_filter.enabled = true;

apm->ApplyConfig(config)
  
 
apm->noise_reduction()->set_level(kHighSuppression);
apm->noise_reduction()->Enable(true);
//处理远端信号
apm->ProcessReverseStream(render_frame);

//实时通信需要设置音频增益、延迟等参数，采集到的数据调用ProcessStream处理
apm->set_stream_delay_ms(delay_ms);
apm->set_stream_analog_level(analog_level);

apm->ProcessStream(capture_frame);

```

AudioProcessing是一个接口类，这个接口类包含了Config字段和通用音频处理方法组成的接口类。

```c++
 class RTC_EXPORT AudioProcessing : public rtc::RefCountInterface {
    public:
  
    // Accepts and produces a ~10 ms frame of interleaved 16 bit integer audio as
    // specified in `input_config` and `output_config`. `src` and `dest` may use
    // the same memory, if desired.
    virtual int ProcessStream(const int16_t* const src,
                                const StreamConfig& input_config,
                                const StreamConfig& output_config,
                                int16_t* const dest) = 0;
   
      // Accepts deinterleaved float audio with the range [-1, 1]. Each element of
      // `src` points to a channel buffer, arranged according to `input_stream`. At
      // output, the channels will be arranged according to `output_stream` in
      // `dest`.
     //
     // The output must have one channel or as many channels as the input. `src`
     // and `dest` may use the same memory, if desired.
     virtual int ProcessStream(const float* const* src,
                               const StreamConfig& input_config,
                               const StreamConfig& output_config,
                               float* const* dest) = 0;
  
     // Accepts and produces a ~10 ms frame of interleaved 16 bit integer audio for
     // the reverse direction audio stream as specified in `input_config` and
     // `output_config`. `src` and `dest` may use the same memory, if desired.
     virtual int ProcessReverseStream(const int16_t* const src,
                                      const StreamConfig& input_config,
                                      const StreamConfig& output_config,
                                      int16_t* const dest) = 0;
  
     // Accepts deinterleaved float audio with the range [-1, 1]. Each element of
     // `data` points to a channel buffer, arranged according to `reverse_config`.
     virtual int ProcessReverseStream(const float* const* src,
                                      const StreamConfig& input_config,
                                      const StreamConfig& output_config,
                                      float* const* dest) = 0;
  
     // Accepts deinterleaved float audio with the range [-1, 1]. Each element
     // of `data` points to a channel buffer, arranged according to
     // `reverse_config`.
     virtual int AnalyzeReverseStream(const float* const* data,
                                      const StreamConfig& reverse_config) = 0;
     virtual int set_stream_delay_ms(int delay) = 0;
     virtual int stream_delay_ms() const = 0;
     static int GetFrameSize(int sample_rate_hz) { return sample_rate_hz / 100; }
  };
```



APM模块有很多处理算法，包括AEC、NS、AGC等，这里以AEC mobile版(PC版本用aec3文件夹下的)本为例，说明其实如何嵌入到APM模块的，首先AEC mobile算法的核心实现是用c代码实现的，位于modules/audio_processing/aecm/文件夹下。有如下几个文件：

```shell
aecm_core.cc           aecm_core.h            aecm_core_c.cc         aecm_core_mips.cc      aecm_core_neon.cc      aecm_defines.h         echo_control_mobile.cc echo_control_mobile.h
```

在EchoControlMobileImpl类init时，会创建canceller对象

```c++
class EchoControlMobileImpl::Canceller {
 public:
  Canceller() {
    state_ = WebRtcAecm_Create();
    RTC_CHECK(state_);
  }

  ~Canceller() {
    RTC_DCHECK(state_);
    WebRtcAecm_Free(state_);
  }

  Canceller(const Canceller&) = delete;
  Canceller& operator=(const Canceller&) = delete;

  void* state() {
    RTC_DCHECK(state_);
    return state_;
  }

  void Initialize(int sample_rate_hz) {
    RTC_DCHECK(state_);
    //这里调用echo_control_mobile.cc文件里的方法，这里的state_定义为void*类型，而在.c算法实现层，
    //则是AecMobile*类型的，在具体使用的地方强制转换下类型即可，这样也实现了隔离。
    //int32_t WebRtcAecm_GetBufferFarendError(void* aecmInst,
    //                                    const int16_t* farend,
    //                                    size_t nrOfSamples) {
    // AecMobile* aecm = static_cast<AecMobile*>(aecmInst);
    // ...
    // }
    int error = WebRtcAecm_Init(state_, sample_rate_hz);
    RTC_DCHECK_EQ(AudioProcessing::kNoError, error);
  }

 private:
  void* state_;
};

void EchoControlMobileImpl::Initialize(int sample_rate_hz,
                                       size_t num_reverse_channels,
                                       size_t num_output_channels) {
  low_pass_reference_.resize(num_output_channels);
  for (auto& reference : low_pass_reference_) {
    reference.fill(0);
  }

  stream_properties_.reset(new StreamProperties(
      sample_rate_hz, num_reverse_channels, num_output_channels));

  // AECM only supports 16 kHz or lower sample rates.
  RTC_DCHECK_LE(stream_properties_->sample_rate_hz,
                AudioProcessing::kSampleRate16kHz);

  cancellers_.resize(
      NumCancellersRequired(stream_properties_->num_output_channels,
                            stream_properties_->num_reverse_channels));

  for (auto& canceller : cancellers_) {
    if (!canceller) {
      //调用上面的Canceller函数创建canceller对象，并初始化该对象
      canceller.reset(new Canceller());
    }
    canceller->Initialize(sample_rate_hz);
  }
  Configure();
}
```

其它的算法和其套路类似，这样在APM类里定义EchoControlMobileImpl成员变量，这样可以使用成员变量调用具体算法了。**AudioProcessingImpl**类的submodules_成员就有`std::unique_prt<EchoControlMobileImpl> echo_control_mobile`这一成员定义。

在capture端会调用config配置的算法完成相应处理。

```c++
int AudioProcessingImpl::ProcessStream(const int16_t* const src,
                                       const StreamConfig& input_config,
                                       const StreamConfig& output_config,
                                       int16_t* const dest) {
  TRACE_EVENT0("webrtc", "AudioProcessing::ProcessStream_AudioFrame");
  RETURN_ON_ERR(MaybeInitializeCapture(input_config, output_config));

  MutexLock lock_capture(&mutex_capture_);
  DenormalDisabler denormal_disabler(use_denormal_disabler_);

  capture_.capture_audio->CopyFrom(src, input_config);
  if (capture_.capture_fullband_audio) {
    capture_.capture_fullband_audio->CopyFrom(src, input_config);
  }
  //ProcessCaptureStreamLocked调用算法的核心处理函数
  RETURN_ON_ERR(ProcessCaptureStreamLocked());
  if (submodule_states_.CaptureMultiBandProcessingPresent() ||
      submodule_states_.CaptureFullBandProcessingActive()) {
    if (capture_.capture_fullband_audio) {
      capture_.capture_fullband_audio->CopyTo(output_config, dest);
    } else {
      capture_.capture_audio->CopyTo(output_config, dest);
    }
  }


  return kNoError;
}
```

int AudioProcessingImpl::ProcessCaptureStreamLocked() 函数有三百多行，精简了和算法调用无关的函数，其主体调用流程如下：

```c++
int AudioProcessingImpl::ProcessCaptureStreamLocked() {
  //先对信号进行
     submodules_.high_pass_filter->Process(capture_buffer,
                                          /*use_split_band_data=*/false);
     if (submodules_.noise_suppressor) {
      submodules_.noise_suppressor->Process(capture_buffer);
    }
  //调用mobile 版本AEC算法
    RETURN_ON_ERR(submodules_.echo_control_mobile->ProcessCaptureAudio(
        capture_buffer, stream_delay_ms()));
  

    if (submodules_.agc_manager) {
      submodules_.agc_manager->Process(capture_buffer);
    }
  
    if (submodules_.echo_detector) {
        submodules_.echo_detector->AnalyzeCaptureAudio(
        rtc::ArrayView<const float>(capture_buffer->channels()[0],
                                      capture_buffer->num_frames()));
    }
  
    if (!!submodules_.voice_activity_detector) {
        voice_probability = submodules_.voice_activity_detector->Analyze(
        AudioFrameView<const float>(capture_buffer->channels(),
                                      capture_buffer->num_channels(),
                                      capture_buffer->num_frames()));
    }
}
```



由上可以看到算法是如何调用的，接下来还剩一个问题，AudioProcessingImpl这个对象的实例上层是如何定义的？任何想使用该类的文件只需要include该头文件`#include "modules/audio_processing/include/audio_processing.h"`在其类中再定义` rtc::scoped_refptr<AudioProcessing> audio_processing;`这样在就可以开发编译代码了，再链接的时候提供该库即可。通常只在WebRtcVoiceEngine中使用，也可以在tranport stream层或者channel层使用，通常还会和audio device module一同存在，因为audio device module采集数据，然后直接调用APM处理，这样的好处是采集是一个线程，每次采集约10ms的数据量，算法也是按照10ms帧长去处理的，这样处理起来紧凑且方便。

关于audio模块之间的组合以及调用见webrtc_voice_engine.h/webrtc_voice_engine.cc
