# ch00 WebRTC架构

https://blog.csdn.net/shichaog/article/details/105864137 原文见，内容比较长，故只罗列部分。

WebRTC是Google开源的Web实时音视频通信框架。其提供P2P的音频、视频和一般数据传输协议栈的支持，其音频主要包括：采集播放、众多音频编解码器、语音增强、回声消除、网络均衡和拥塞控制等音频处理单元，其视频主要包括：采集播放，丢包隐藏，视频增强和编解码几个部分，支持的编解码有H264、VP8，VP9，AV1、H265在2020年4月已经基本集成完毕；在网络方面WebRTC提供针对音视频的动态抖动buffer管理和丢包隐藏处理，另外也提供基于STURN和TRUN的P2P的多媒体数据传输。
  理解WebRTC Native层需要非常多的知识，包括各种网络协议，网络通信，音频视频编解码，音视频处理，音视频数据采集和播放，C/S(client service)通信架构以及多平台等，具体到编程实现上，包括object c 、c/c++，java，涉及平台包括MAC、IOS、Linux、windows以及android，由于官方WebRTC给的nija编译架构，这一编译系统屏蔽了较多的细节，这使得基于vs2019、Xcode以及androidstudio和GNU make编译需要重新搞一遍，很多新手在编译WebRTC是遇到的难题就退缩了。

本篇文章以Native层为出发点，对于Web浏览器不涉及，WebRTC Natvie代码的核心部分是音频、视频以及信令三个部分。为了便于理解这里以WebRTC自带的例子入手，涉及到Ubuntu和IOS两个平台的编译，例子在exmaple目录文件夹下。视频会议服务端架构

在视频会议场景中，主要有三种类型的服务器架构，Mesh，MCU和SFU，WebRTC主推的是无中心的P2P架构，会为每一个端建立一个PeerConnection对象，这就是Mesh架构。

![img](/Users/weijiez/webrtc/WebRTC_server_arch.png)

Mesh架构由于不需要多媒体（音视频）服务器，因而成本是最低的，安全性好，然而当人数较多时，可以看到P2P的链接数和带宽需求量变大，这在人数较少（如5人之内）是较为实用的，MCU架构需要服务器进行视频的解码、转码、混合和编码，但是上述视频的处理是比较消耗服务器资源的，因而成本较高，且会引入通信延迟，但是由于连接数量少，带宽压力小，因而在参会人数在数十人的场景中使用到，SFU架构和MCU架构一样都需要中心节点服务器，不同的是改服务器只负责转发，不负责视频处理，这在上百人场景中会用到该架构。
STURN & TRUN &ICE
在上述的Mesh架构下，需要进行透传以便通信双方能够正常进行，这就使用到的STURN和TURN以及ICE技术。

WebRTC的信令传输可以慢些但需要可靠，而流媒体则实时性优，可以存在丢包、错包；这就意味着需要两套网络传输协议，一套是基于TCP的可靠传输协议用于信令等传输，一套是基于UDP的TRP协议用于实时多媒体数据的传输。其协议栈如下图所示：
![img](/Users/weijiez/webrtc/WebRTC_arch_01.png)

ICE（Interactive Connectivity Establishment）协议(RFC 5245)

STUN（Session Traversal Utilities for NAT）(RFC5389)

TURN（Traversal Using Relays around NAT）(RFC 5766)

这三个协议是基于UDP协议建立和维持P2P连接的必要网络组件；DTLS（Datagram Transport Protocol）用于P2P双方数据安全传输。

## WebRTC代码架构

WebRTC目录结构如下所示，其中紫色的为目录。

![img](/Users/weijiez/webrtc/webrtc_arch_02.png)

各个目录的功能如下：

api目录：是对WebRTC功能件的封装，以更方便应用层调用，这里封装的内容包括audio、video、数据通道以及RTP传输，并在create_peerconnection_factory.h文件中定义了P2P通信的核心类PeerConnectionFactoryInterface；

audio目录：这里的audio层是用于发送和接收音频数据流的网络层，真实硬件的采集播放放在adm（audio device module），增强处理放在apm(audio processing module)里，adm和apm并不在这一目录下；

base目录：提供了一些依赖OS的基础函数，比如内存管理等；

build相关目录：使用与编译WebRTC的，在编译小节中会有编译说明；

call目录：从字面可以知道是用于通信用的，主要是RTP和RTCP相关协议的封装一遍WebRTC使用；

common_audio和common_video目录：音视频的各种算法都可能用到的，比如fir滤波，环形缓冲区，窗函数等；

examples：P2P等各个平台各种例子所在的目录；
media：是对video和audio增强和编解码的封装层，即video engine和audio engine。

modules：音视频具体功能实现所在的目录，如音视频编解码实现；音频混音、处理以及设备管理，视频采集播放以及数据发送和占用带宽估计等；

pc(peer connection)目录：P2P连接实现的核心目录;

sdk目录：android平台应用层Java和MAC平台应用层Object C访问natvie层的桥接层；

server端
STUN服务器：用于获取设备的外网地址；

TURN服务器：在P2P通信失败后用于中继；

ICE框架，整合了TURN和STUN。

信令服务器：负责端到端的连接，如SDP，candidate等；
因为由于IPv4地址数量不够用和安全的问题，开会的双方基本都在防火墙和NAT之后，通过运营商接入公网，当在家时手机、电脑、网络电视通过电信路由器上网时，路由器分配给我们的地址就是“192.168.XXX.XXX”，但是在公网上我们的数据头地址被转换成电信服务商提供的地址了，如果双发希望直接通信而不需要公共服务器中转（加大了延迟和丢包的不确定性）数据包，这时需要NAT穿透技术，STUN和TURN就是这种透传协议，ICE是一套整合了这两个协议的框架。

client端
Android IOS Windows MAC Linux 浏览器。
![img](/Users/weijiez/webrtc/webrtc_arch_03.png)

由于不同平台使用了不同的UI库和编程语言，他们的实现差异很大，但是不同的平台都会支持c/c++，所以为了适配不同的平台，WebRTC提供了SDK层，Linux平台基于使用GTK的c++，MAC和IOS平台基于使用cocoa库的Object c，android平台基于JAVA，为了让这些平台都能够调用c/c++核心函数，WebRTC提供了android和object c的封装层，这称为SDK层，android中使用了JNI机制使得UI层的JAVA程序和实现核心功能的c/c++程序可以互相调用，类似的object c使用了.mm扩展程序是得UI的.m程序可以和c/c++互相调用。由于访问各个平台都提供了C API（为了效率），所有已在音视频以及网络API都可以直接通过包含不同操作系统的头文件来实现跨平台差异化编译。

