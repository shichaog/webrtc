# 编码架构和实现

ACM是audio coding module的简称。

WebRTC 的audio coding 模块可以处理音频接收和发送，`acm2`目录是接收和发送的API实现。每个音频发送帧使用包含时长为10ms音频数据，通过`Add10MsData()`接口提供给音频编码模块，音频编码模块使用对应的编码器对音频帧进行编码，并将编码好的数据传给预先注册的音频分组回调，该回调将编码的音频打包成RT包，并通过传输层发送出去，WebRTC内置的音频编码器包括G711、G722, ilbc, isac, opus,pcm16b等，音频网络适配器为音频编码器（目前仅限于OPU）提供附加功能，以使音频编码器适应网络条件（带宽、丢包率等）。音频接收包通过`IncomingPacket()`实现，接收到的数据包将有jitter buffer(NetEq)模块处理，处理的内容包括解码，音频解码器通过解码器工厂类创建，解码后的数据通过`PlayData10Ms()`获取。

## 编码模块接口类
这里的接口包括编码和解码两个部分，

音频编码接口类的核心内容如下：

```c
// modules/audio_coding/include/audio_coding_module.h 
 30 // forward declarations
 31 class AudioDecoder;
 32 class AudioEncoder;
 33 class AudioFrame;
 34 struct RTPHeader;
 62 class AudioCodingModule {
 66  public:
 67   struct Config {
 68     explicit Config(
 69         rtc::scoped_refptr<AudioDecoderFactory> decoder_factory = nullptr);
 70     Config(const Config&);
 71     ~Config();
 72
 73     NetEq::Config neteq_config;
 74     Clock* clock;
   //工厂类创建解码器
 75     rtc::scoped_refptr<AudioDecoderFactory> decoder_factory;
   //工厂类创建NetEq
 76     NetEqFactory* neteq_factory = nullptr;
 77   };
 //这是设计模式中类和对象的创建方法，即通过这个static 方法创建  
 79   static AudioCodingModule* Create(const Config& config);
 
  //这里定义成了纯虚函数，这种是接口类的常用方法，纯虚函数的好处是子类必须实现
  //这些类对应的方法，否则编译报错。
 136   virtual int32_t Add10MsData(const AudioFrame& audio_frame) = 0;
 172   virtual int32_t InitializeReceiver() = 0;
 192   virtual int32_t IncomingPacket(const uint8_t* incoming_payload,
 193                                  size_t payload_len_bytes,
 194                                  const RTPHeader& rtp_header) = 0;
 216   virtual int32_t PlayoutData10Ms(int32_t desired_freq_hz,
 217                                   AudioFrame* audio_frame,
 218                                   bool* muted) = 0;
 }
```

这里使用工厂类设计模式，是对编解码接口类的实现，这一接口对大部分实时音视频频应用场景，并不需要做更改，和编码相比，接收的待解码数据是通过网络传输的，这会导致丢包、抖动以及延迟到达、乱序等问题，所以需要实现jitter buffer功能，因而AudioCodingModuleImpl在实现编码接口类AudioCodingModule方法的时候，定义了AcmReceiver（NetEq以及解码）成员，接收到的数据被送到这个模块去解码了。

```c++
 //acm2/audio_coding_module.cc
 42 class AudioCodingModuleImpl final : public AudioCodingModule {
   //override是c++中重写关键词，即其集成的父类中必须有Add10MsData这个虚函数定义，否则编译报错
 58   // Add 10 ms of raw (PCM) audio data to the encoder.
 59   int Add10MsData(const AudioFrame& audio_frame) override;
 72   // Initialize receiver, resets codec database etc.
 73   int InitializeReceiver() override;
 77   // Incoming packet from network parsed and ready for decode.
 78   int IncomingPacket(const uint8_t* incoming_payload,
 79                      const size_t payload_length,
 80                      const RTPHeader& rtp_info) override;
 82   // Get 10 milliseconds of raw audio data to play out, and
 83   // automatic resample to the requested frequency if > 0.
 84   int PlayoutData10Ms(int desired_freq_hz,
 85                       AudioFrame* audio_frame,
 86                       bool* muted) override;
   
 98  private:
 128   int Add10MsDataInternal(const AudioFrame& audio_frame, InputData* input_data)
 129       RTC_EXCLUSIVE_LOCKS_REQUIRED(acm_mutex_);
 130
 131   // TODO(bugs.webrtc.org/10739): change `absolute_capture_timestamp_ms` to
 132   // int64_t when it always receives a valid value.
 133   int Encode(const InputData& input_data,
 134              absl::optional<int64_t> absolute_capture_timestamp_ms)
   
 137   int InitializeReceiverSafe() RTC_EXCLUSIVE_LOCKS_REQUIRED(acm_mutex_);
   
 162   rtc::Buffer encode_buffer_ RTC_GUARDED_BY(acm_mutex_);
 163   uint32_t expected_codec_ts_ RTC_GUARDED_BY(acm_mutex_);
 164   uint32_t expected_in_ts_ RTC_GUARDED_BY(acm_mutex_);
 165   acm2::ACMResampler resampler_ RTC_GUARDED_BY(acm_mutex_);
 166   acm2::AcmReceiver receiver_;  // AcmReceiver has it's own internal lock.
 }
//这是设计模式一种思想，通过Create方法创建实现，但返回类型是接口类型，这样实现了接口和实现的隔离，当接口类和实现不在同一个库中时，可以做到只需要直接连接重新编译的实现，而接口类所在库不用再次编译，开发上进行了隔离。
209 AudioCodingModuleImpl::AudioCodingModuleImpl(
210     const AudioCodingModule::Config& config)
211     : expected_codec_ts_(0xD87F3F9F),
212       expected_in_ts_(0xD87F3F9F),
213       receiver_(config),
214       bitrate_logger_("WebRTC.Audio.TargetBitrateInKbps"),
215       encoder_stack_(nullptr),
216       previous_pltype_(255),
217       receiver_initialized_(false),
218       first_10ms_data_(false),
219       first_frame_(true),
220       packetization_callback_(NULL),
221       codec_histogram_bins_log_(),
222       number_of_consecutive_empty_packets_(0) {
223   if (InitializeReceiverSafe() < 0) {
224     RTC_LOG(LS_ERROR) << "Cannot initialize receiver";
225   }
226   RTC_LOG(LS_INFO) << "Created";
227 }
633 AudioCodingModule* AudioCodingModule::Create(const Config& config) {
634   return new AudioCodingModuleImpl(config);
635 }

```

### 编码数据流

接收到的数据包经过合法性、按需混音等操作后，发送给注册号的编码模块编码，这里集中于解码数据流，encoder_stack_是一个AudioEncoder类的一个智能指针。

```c++

231 int32_t AudioCodingModuleImpl::Encode(
232     const InputData& input_data,
233     absl::optional<int64_t> absolute_capture_timestamp_ms) {
234   // TODO(bugs.webrtc.org/10739): add dcheck that
235   // `audio_frame.absolute_capture_timestamp_ms()` always has a value.
236   AudioEncoder::EncodedInfo encoded_info;
237   uint8_t previous_pltype;
264   encoded_info = encoder_stack_->Encode(
265       rtp_timestamp,
266       rtc::ArrayView<const int16_t>(
267           input_data.audio,
268           input_data.audio_channel * input_data.length_per_channel),
269       &encode_buffer_);
}

334 // Add 10MS of raw (PCM) audio data to the encoder.
335 int AudioCodingModuleImpl::Add10MsData(const AudioFrame& audio_frame) {
336   MutexLock lock(&acm_mutex_);
//做声道数、采样率等合法性检查，适当的混音
337   int r = Add10MsDataInternal(audio_frame, &input_data_);
338   // TODO(bugs.webrtc.org/10739): add dcheck that
339   // `audio_frame.absolute_capture_timestamp_ms()` always has a value.
340   return r < 0
341              ? r
342              : Encode(input_data_, audio_frame.absolute_capture_timestamp_ms());
343 }

```
Encode是提供给上层的接口类，实际内部调用protected类型的EncodeImpl()实现编码，因而每个编解码器（opus、pcm16、g711）必须实现该方法。

### 收包解码数据流

相比编码而言，因实时音频引用一般要求的网络延迟在300ms以内，因而收包要做抖动处理，而解码正是由处理抖动的模块调用的，所以使用了AcmReceiver这个类定义了接受模块公共的内容。

```c++
559 // Incoming packet from network parsed and ready for decode.
560 int AudioCodingModuleImpl::IncomingPacket(const uint8_t* incoming_payload,
561                                           const size_t payload_length,
562                                           const RTPHeader& rtp_header) {
563   RTC_DCHECK_EQ(payload_length == 0, incoming_payload == nullptr);
564   return receiver_.InsertPacket(
565       rtp_header,
566       rtc::ArrayView<const uint8_t>(incoming_payload, payload_length));
567 }
```

### 获取解码数据流

解码的数据流在NetEq模块中，有实时性的概念和要求在这里，因而直接调用receiver_类的方法获取数据。

```c++
569 // Get 10 milliseconds of raw audio data to play out.
570 // Automatic resample to the requested frequency.
571 int AudioCodingModuleImpl::PlayoutData10Ms(int desired_freq_hz,
572                                            AudioFrame* audio_frame,
573                                            bool* muted) {
574   // GetAudio always returns 10 ms, at the requested sample rate.
575   if (receiver_.GetAudio(desired_freq_hz, audio_frame, muted) != 0) {
576     RTC_LOG(LS_ERROR) << "PlayoutData failed, RecOut Failed";
577     return -1;
578   }
579   return 0;
580 }
```

接下来看实现编码和接收的AudioEncoder类和AcmReceiver类。

### AudioEncoder类

AudioEncoder作为编码器的接口类，其定义于api/audio_codecs/audio_encoder.h，编码类是一个通用的类型，因为Opus、G711等具体实现会继承这个类，因而这个类定义了编码器一些公共的内容，比如编码比特率、编码、FEC、DTX等，此外由于是实时场景，所以网络情况会影响编码器最优参数的选择，同样的这里忽略网络统计相关的实现。

```c++
 64 // This is the interface class for encoders in AudioCoding module. Each codec
 65 // type must have an implementation of this class.
 66 class AudioEncoder {
 67  public:
 68   // Used for UMA logging of codec usage. The same codecs, with the
 69   // same values, must be listed in
 70   // src/tools/metrics/histograms/histograms.xml in chromium to log
 71   // correct values.
 72   enum class CodecType {
 73     kOther = 0,  // Codec not specified, and/or not listed in this enum
 74     kOpus = 1,
 75     kIsac = 2,
 76     kPcmA = 3,
 77     kPcmU = 4,
 78     kG722 = 5,
 79     kIlbc = 6,
 80
 81     // Number of histogram bins in the UMA logging of codec types. The
 82     // total number of different codecs that are logged cannot exceed this
 83     // number.
 84     kMaxLoggedAudioCodecTypes
 85   };
   
 144   // Accepts one 10 ms block of input audio (i.e., SampleRateHz() / 100 *
 145   // NumChannels() samples). Multi-channel audio must be sample-interleaved.
 146   // The encoder appends zero or more bytes of output to `encoded` and returns
 147   // additional encoding information.  Encode() checks some preconditions, calls
 148   // EncodeImpl() which does the actual work, and then checks some
 149   // postconditions.
 150   EncodedInfo Encode(uint32_t rtp_timestamp,
 151                      rtc::ArrayView<const int16_t> audio,
 152                      rtc::Buffer* encoded);
   
 154   // Resets the encoder to its starting state, discarding any input that has
 155   // been fed to the encoder but not yet emitted in a packet.
 156   virtual void Reset() = 0;
 157
 158   // Enables or disables codec-internal FEC (forward error correction). Returns
 159   // true if the codec was able to comply. The default implementation returns
 160   // true when asked to disable FEC and false when asked to enable it (meaning
 161   // that FEC isn't supported).
 162   virtual bool SetFec(bool enable);
 163
 164   // Enables or disables codec-internal VAD/DTX. Returns true if the codec was
 165   // able to comply. The default implementation returns true when asked to
 166   // disable DTX and false when asked to enable it (meaning that DTX isn't
 167   // supported).
 168   virtual bool SetDtx(bool enable);
 169
 170   // Returns the status of codec-internal DTX. The default implementation always
 171   // returns false.
 172   virtual bool GetDtx() const;
 174   // Sets the application mode. Returns true if the codec was able to comply.
 175   // The default implementation just returns false.
 176   enum class Application { kSpeech, kAudio };
 177   virtual bool SetApplication(Application application);
 //在接口类中调用子类的编码具体实现，以实现用同一个接口调用不同的编码器
 //这是每一个不同类型编码器必须实现的接口方法
  protected:
  // Subclasses implement this to perform the actual encoding. Called by
  // Encode().
  virtual EncodedInfo EncodeImpl(uint32_t rtp_timestamp,
                                 rtc::ArrayView<const int16_t> audio,
                                 rtc::Buffer* encoded) = 0;
 }
```

这个接口类编码API `Encode`的实现位于api/audio_codecs/audio_encoder.cc

```c++
//编码的信息存放于EncodedInfo
AudioEncoder::EncodedInfo AudioEncoder::Encode(
    uint32_t rtp_timestamp, //rtp时间戳的作用是用于音视频播放和同步
    rtc::ArrayView<const int16_t> audio, //带编码PCM数据
    rtc::Buffer* encoded) {//编码后的数据
  TRACE_EVENT0("webrtc", "AudioEncoder::Encode");
  RTC_CHECK_EQ(audio.size(),
               static_cast<size_t>(NumChannels() * SampleRateHz() / 100));

  const size_t old_size = encoded->size();
  EncodedInfo info = EncodeImpl(rtp_timestamp, audio, encoded);
  RTC_CHECK_EQ(encoded->size() - old_size, info.encoded_bytes);
  return info;
}
```

#### Opus编码器类实现

opus第三方库开源实现都是基于c代码的，开源的第三方库都放在src/third_party目录下，这样做的好处是将第三方库和webrtc实现隔离，当第三方库没有改动时并不需要重新编译，对单个第三库而言节约的时间也许并不多，但是当第三库很多时，编译还是很耗时间的，使用这种解耦设计可以大大提高开发效率。

```c++
// modules/audio_coding/codecs/opus/audio_encoder_opus.h

 32 class AudioEncoderOpusImpl final : public AudioEncoder {
 33  public:
 136  protected:
 137   EncodedInfo EncodeImpl(uint32_t rtp_timestamp,
 138                          rtc::ArrayView<const int16_t> audio,
 139                          rtc::Buffer* encoded) override;
 141  private:
 146   static void AppendSupportedEncoders(std::vector<AudioCodecSpec>* specs);
 185   std::vector<int16_t> input_buffer_;
 186   OpusEncInst* inst_;
 199   friend struct AudioEncoderOpus;
 }
```

其实现位于

```c++
// modules/audio_coding/codecs/opus/audio_encoder_opus.cc
648 AudioEncoder::EncodedInfo AudioEncoderOpusImpl::EncodeImpl(
649     uint32_t rtp_timestamp,
650     rtc::ArrayView<const int16_t> audio,
651     rtc::Buffer* encoded) {
652   MaybeUpdateUplinkBandwidth();
654   if (input_buffer_.empty())
655     first_timestamp_in_buffer_ = rtp_timestamp;
656
657   input_buffer_.insert(input_buffer_.end(), audio.cbegin(), audio.cend());
658   if (input_buffer_.size() <
659       (Num10msFramesPerPacket() * SamplesPer10msFrame())) {
660     return EncodedInfo();
661   }
//调用WebRtcOpus_Encode完成具体编码，这个函数实现于opus_interface.cc,interface.cc就是对第三方opus库函数的封装和调用，其主要是调用opus_encode或opus_multistream_encode编码。
665   const size_t max_encoded_bytes = SufficientOutputBufferSize();
666   EncodedInfo info;
667   info.encoded_bytes = encoded->AppendData(
668       max_encoded_bytes, [&](rtc::ArrayView<uint8_t> encoded) {
669         int status = WebRtcOpus_Encode(
670             inst_, &input_buffer_[0],
671             rtc::CheckedDivExact(input_buffer_.size(), config_.num_channels),
672             rtc::saturated_cast<int16_t>(max_encoded_bytes), encoded.data());
673
674         RTC_CHECK_GE(status, 0);  // Fails only if fed invalid data.
675
676         return static_cast<size_t>(status);
677       });
678   input_buffer_.clear();
679
680   bool dtx_frame = (info.encoded_bytes <= 2);
681
  
682   // Will use new packet size for next encoding.
683   config_.frame_size_ms = next_frame_length_ms_;
684
685   if (adjust_bandwidth_ && bitrate_changed_) {
686     const auto bandwidth = GetNewBandwidth(config_, inst_);
687     if (bandwidth) {
688       RTC_CHECK_EQ(0, WebRtcOpus_SetBandwidth(inst_, *bandwidth));
689     }
690     bitrate_changed_ = false;
691   }
```
上文说到这个类是每个编码器都要实现的，在比如ilbc编码的实现如下：
```c++
AudioEncoder::EncodedInfo AudioEncoderIlbcImpl::EncodeImpl(
    uint32_t rtp_timestamp,
    rtc::ArrayView<const int16_t> audio,
    rtc::Buffer* encoded) {
  // Save timestamp if starting a new packet.
  if (num_10ms_frames_buffered_ == 0)
    first_timestamp_in_buffer_ = rtp_timestamp;

  // Buffer input.
  std::copy(audio.cbegin(), audio.cend(),
            input_buffer_ + kSampleRateHz / 100 * num_10ms_frames_buffered_);

  // If we don't yet have enough buffered input for a whole packet, we're done
  // for now.
  if (++num_10ms_frames_buffered_ < num_10ms_frames_per_packet_) {
    return EncodedInfo();
  }

  // Encode buffered input.
  RTC_DCHECK_EQ(num_10ms_frames_buffered_, num_10ms_frames_per_packet_);
  num_10ms_frames_buffered_ = 0;
  //同样的其调用第三方库中的WebRtcIlbcfix_Encode去实现具体的编码
  size_t encoded_bytes = encoded->AppendData(
      RequiredOutputSizeBytes(), [&](rtc::ArrayView<uint8_t> encoded) {
        const int r = WebRtcIlbcfix_Encode(
            encoder_, input_buffer_,
            kSampleRateHz / 100 * num_10ms_frames_per_packet_, encoded.data());
        RTC_CHECK_GE(r, 0);

        return static_cast<size_t>(r);
      });

  RTC_DCHECK_EQ(encoded_bytes, RequiredOutputSizeBytes());

  EncodedInfo info;
  info.encoded_bytes = encoded_bytes;
  info.encoded_timestamp = first_timestamp_in_buffer_;
  info.payload_type = payload_type_;
  info.encoder_type = CodecType::kIlbc;
  return info;
}
```

至此可以看到具体的编码器是如何通编码器接口类AudioEncoder派生出来的了，那还有一个问题，就是这些派生出来的编码器类是如何最终被应用程序调用的？
这涉及到call、stream、channel的概念了。
channel包括send和receive两种，channel是和ssrc挂钩的，用于表示来源，比如在视频通信时，发送的可能是麦克风采集的声音，也可以是分享的ppt上播放的声音，这两种声音就是通过ssrc来区分的，对于接收是一个道理，同时接收多个人会议音频时，ssrc也是不同的，每一个ssrc对应于一个channel，因为每一路处理的方式可能并不一样，比如麦克风采集信号要做语音增强，但是ppt分享的声音则不需要，channel主要是编码器和rtp的封装以及端到端加密等信息。
stream也分发送和接收，属于transport层，即对channel层打包好的rtp/rtcp包进行发送和接收。
call的概念就是双向通话会议了，也许有些实现命名不一样，但是不论是双人还是对人会议，通话这个概念是必须有的。这个call的类就是peer connection例子使用的通话类。
```c+++
//pc/peer_connection.h
  // Creates a PeerConnection and initializes it with the given values.
  // If the initialization fails, the function releases the PeerConnection
  // and returns nullptr.
  //
  // Note that the function takes ownership of dependencies, and will
  // either use them or release them, whether it succeeds or fails.
  static RTCErrorOr<rtc::scoped_refptr<PeerConnection>> Create(
      rtc::scoped_refptr<ConnectionContext> context,
      const PeerConnectionFactoryInterface::Options& options,
      std::unique_ptr<RtcEventLog> event_log,
      std::unique_ptr<Call> call,
      const PeerConnectionInterface::RTCConfiguration& configuration,
      PeerConnectionDependencies dependencies);
```
对channel、stream以及call的细节实现这里不展开，待到具体层再分析。至此，AudioEncoder类实现和使用流程已结束，接下来是jitter buffer的接收类的功能和实现。
### AcmReceiver
这里再罗列一下third_party/webrtc/modules/audio_coding/acm2/audio_coding_module.cc中调用AudioCodingModuleImpl中实现接收的一些方法。
```
 acm2::AcmReceiver receiver_;  // AcmReceiver has it's own internal lock.
 
 receiver_.SetCodecs(codecs);
 receiver_.InsertPacket(
      rtp_header,
      rtc::ArrayView<const uint8_t>(incoming_payload, payload_length));
 receiver_.GetAudio(desired_freq_hz, audio_frame, muted);
 receiver_.GetNetworkStatistics(statistics);
```
AcmRecevier的主要方法就是上述代码段417-422中调用的，而InsertPacket和GetAudio这两个接收数据流内部调用了neteq_同名方法，其创建的方式如下，主要是调用neteq工程类创建该对象，并且将其和解码器关联。
```c++
  37 std::unique_ptr<NetEq> CreateNetEq(
  38     NetEqFactory* neteq_factory,
  39     const NetEq::Config& config,
  40     Clock* clock,
  41     const rtc::scoped_refptr<AudioDecoderFactory>& decoder_factory) {
  42   if (neteq_factory) {
  43     return neteq_factory->CreateNetEq(config, decoder_factory, clock);
  44   }
  45   return DefaultNetEqFactory().CreateNetEq(config, decoder_factory, clock);
  46 }
  
  50 AcmReceiver::AcmReceiver(const AudioCodingModule::Config& config)
  51     : last_audio_buffer_(new int16_t[AudioFrame::kMaxDataSizeSamples]),
  52       neteq_(CreateNetEq(config.neteq_factory,
  53                          config.neteq_config,
  54                          config.clock,
  55                          config.decoder_factory)),
  56       clock_(config.clock),
  57       resampled_last_output_frame_(true) {
  58   RTC_DCHECK(clock_);
  59   memset(last_audio_buffer_.get(), 0,
  60          sizeof(int16_t) * AudioFrame::kMaxDataSizeSamples);
  61 }
  
```
neteq_也是通过工厂类的方法创建，NetEqFactory有两种，一种是default一种是customer，NetEq是一个接口类，这个类返回的是NetEq对象，
```c++
// Creates NetEq instances using the settings provided in the config struct.
class NetEqFactory {
 public:
  virtual ~NetEqFactory() = default;

  // Creates a new NetEq object, with parameters set in `config`. The `config`
  // object will only have to be valid for the duration of the call to this
  // method.
  virtual std::unique_ptr<NetEq> CreateNetEq(
      const NetEq::Config& config,
      const rtc::scoped_refptr<AudioDecoderFactory>& decoder_factory,
      Clock* clock) const = 0;
};
```
WebRTC在实现NetEqFactory的时候，在这个基础之上又封装了两个factory类，
```c++
class CustomNetEqFactory : public NetEqFactory {
 private:
  std::unique_ptr<NetEqControllerFactory> controller_factory_;
}
和
class DefaultNetEqFactory : public NetEqFactory {

 private:
  const DefaultNetEqControllerFactory controller_factory_;
}
```
这两个类中private字段是比NetEqFactory类多出来的，大多数情况下用default的就可以了，除非针对自己的应用场景需要调节相关内容，才真正需要自己customer。之后就是调用NetEQ算法细节内容见《实时语音处理实践指南》第11章，11.2小节。






