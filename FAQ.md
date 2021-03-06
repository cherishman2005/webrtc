
# FAQ

## volume 视频音量设置

将视频音量设置为 20%：
```javascript
myVid = document.getElementById("video1");
myVid.volume = 0.2;
```

## h264分辨率和码率对应关系

h264编码推荐的分辨率和码率关系如下:

| 分辨率      | 码率                 | 
| --------- |  --------------------------- |
| 320x240               |     200-384kbps          | 
| 640x480               |     768-1024kbps         |
| 1280x720(720p)        |     2048-3072kbps        |
| 1920x1080(1080p)      |     5120-8192kbps        |


## VP9

VP9的目标之一是在保证相同质量的情况下相对于h.264可以减少50%左右的码率。

## webrtc统计数据

chrome://webrtc-internals/

![webrtc分辨率和帧率](/img/wbrtc_frames.png)

分辨率：640*480 帧率：25


## Linux: 查看二进制文件 

查看二进制文件，用od或hexdump命令。

$ od -tx1 -tc -Ax binFile
000000  46  4c  56  01  01  00  00  00  09  00  00  00  00  67  42  c0
         F   L   V 001 001  \0  \0  \0  \t  \0  \0  \0  \0   g   B 300
000010  1f  43  23  50  14  07  a4  03  c2  21  1a  80  68  48  e3  c8
       037   C   #   P 024  \a 244 003 302   ! 032 200   h   H 343 310
000020  65  b4  00  01  80  00  02  7e  38  cd  f8  b2  72  c4  80  00
         e 264  \0 001 200  \0 002   ~   8 315 370 262   r 304 200  \0

-tx1选项表示将文件中的字节以十六进制的形式列出来，每组一个字节（类似hexdump的-c选项）
-tc选项表示将文件中的ASCII码以字符形式列出来（和hexdump类似，输出结果最左边的一列是文件中的地址，默认以八进制显示）
-Ax选项要求以十六进制显示文件中的地址

## google performance

- [google performance](https://medium.com/@marielgrace/how-to-analyze-runtime-performance-google-devtools-99fda64c09cb)

# 学习链接

- [淘宝直播技术干货：高清、低延时的实时视频直播技术解密](http://www.52im.net/thread-3220-1-1.html)
