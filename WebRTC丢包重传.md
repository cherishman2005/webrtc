# 概述

WebRTC之所以可以优秀的完成音视频通讯，和它本身的丢包重传机制是密不可分的，今天我们就来看看其中的奥秘。

本文以M76版本展开，如果你的工程是基于其他版本开发的，也可以参考。

## NACK

说到丢包重传就不得不提到NACK技术，那么NACK是什么呢。它的全称是Negative Acknowledgment Packet，意思是否定确认包，说到这里我们应该可以联想到ACK（Acknowledgment Packet，确认包）。没错，二者的意思是相反的。ACK表示通知对方我收到了你发给我的数据包，NACK表示通知对方我没有收到你发给我的数据包。

那么问题来了，为什么会导致对方明明发送了响应的数据包，而我没有收到呢？其中的原因有很多，比如网络问题，因为中间路由器转发丢失，延时较大导致被NACK（可能数据包还在传输中，只是到达时间比较久）等。

基于上述原因，NACK的存在是非常有必要的。它能够及时的通知发送端重传相应的数据包，保证接收端音频和视频的正常播放。NACK其实是RTCP包的一种，用来是对 RTP 数据传输层进行反馈，它包类型是 205。

### 问题一、数据包真丢了，会一直重传吗？
答案是否定的。会有哪些决定因素呢？首先看最大重传次数，源码中默认是10次。意思是`如果相同seq_num的数据包被重传了10次，接收端依然没收到，就不再继续请求重传了`。

```
const int kMaxNackRetries = 10;
```

处理方式是将该数据包从重传列表中移除，具体看源码：



基于问题一，于是我们引入了问题二。

### 问题二、重传次数不到最大限制次数，就会一直等待吗？

很不幸，答案是肯定的。NACK技术作为WebRTC对抗弱网的核心技术之一，有两种发送模式，一种是基于包序列号的发送，一种是基于时间序列的发送。 对于一个包因为不连续而被判为丢失后，接收端会主动请求重传这个数据包。正常情况下，发送端收到通知后，一般是从发送缓存列表中找到这个包并重新发送。可是，如果因为特定的原因，导致上次的RTT（往返延时）很大。源码中默认是100ms。

```
const int kDefaultRttMs = 100;
```

这里需要说明一点，因为WebRTC判断某个包是否超时需要重传的标准是上次的RTT时间。如果当前等待的数据包时间已经大于RTT了，就认为丢了，从而请求重传。如果小于RTT，就继续等待。那么漏洞出来了，如果上次的RTT很大很大，WebRTC确实会等待，但是出现这种情况的概率是很低的。同时，WebRTC还可以通过其他机制避免出现类似的问题，于是引出了问题三。

### 问题三、当大量丢包时，会全部重传吗？

答案是否定的。因为WebRTC不仅限制了重传包的次数，而且还限制了重传包的个数。WebRTC每次要求重传包的个数默认是1000个。
```
const int kMaxNackPackets = 1000;
```

如果丢失的包数量超过1000个，会循环清空 nack_list_ 中关键帧之前的包，直到其长度小于 1000。也就是通过放弃对关键帧首包之前的老旧包的重传请求，直接而快速的请求新近的数据包，相关逻辑源码：


那是不是nack_list_填充到1000个才会请求重传呢？当然不是，只要最大个数不超过1000个，就可以按照kProcessIntervalMs时间间隔请求一次重传包。即使，只丢失了一个包也会在规定的时间进行重传请求。

```
const int kProcessFrequency = 50; 
const int kProcessIntervalMs = 1000 / kProcessFrequency;
```

其中，kProcessIntervalMs = 20ms。这里有些不理解，为什么不直接初始化20ms，却要通过1000/50计算？

## NACK改进

这里需要说明WebRTC新引入的一种机制：NACK延时发送机制，通过控制NACK延时发送的时间间隔，避免固定延时网络下无必要的重传请求。比如，如果kDefaultSendNackDelayMs=20ms，如果因为网络的固有延时，造成某些数据包迟到了10ms，而此时没有NACK延时发送机制的话，这些包都会被认为丢了，从而对这些包请求重传。但是如果有20ms的NACK延时发送，这些包就不会被计算为丢失，从而避免了没有必要的重传请求，避免了资源浪费。
```
const int kDefaultSendNackDelayMs = 0;
```

而且，kDefaultSendNackDelayMs参数值是支持自定义设置的，通过接口传参field_trial设置WebRTC-SendNackDelayMs的值，WebRTC在底层会解析这个参数，如果没有设置的话，默认是10ms。相关逻辑如下：


## 总结
1. WebRTC中丢包重传逻辑是非常复杂的。

1. WebRTC丢包严重时，如超过1000个包时，直接采用PLI去请求关键帧。
   —— 这个就决定了webrtc是半可靠通道。容忍丢失数据后，直接去请求关键帧。

   * 可靠通道：
     ** TCP可靠通道；
     ** 基于UDP的quic可靠通道；或自定协议的可靠通道；

   * 半可靠通道：
     ** 基于UDP的webrtc（rtp/rtcp）的半可靠通道；或自定协议的半可靠通道；

    【注】音视频实时互动常使用半可靠通道
          * 基于UDP的RTP，或自定义协议传输媒体流；
          * 基于UDP的RTCP，或基于TCP的自定义协议作为控制信令；
    

1. 实时互动
   * 基于TCP的RTMP是可以做实时互动的，只是对网络抖动，时延等比较敏感；
     —— 实现相对简单。

   * 最佳方案：采用UDP做实时互动解决方案，采用拥塞控制，NACK/FEC，码率控制等抗网络抖动，低时延。