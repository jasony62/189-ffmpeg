实现实时给视频流添加水印

# 需求
直播场景中通过ffmpeg实现给视频流添加水印。

# 整体方案
ffmpeg从输入端获得视频流，解码（decode）后传递给滤镜（filter），添加图片作为水印，编码（encode）后发送。

为了便于和其他系统对接，输入端和输出端都采用RTP传送。输入为一个SDP文件（input.sdp）,输出端为一个rtp地址。

解码、滤镜、编码是一个串行的过程，但是如果直接用串行的方式处理会产生极大的性能问题。因为视频流的发送端是按照一定频率发送数据，如果每接收1帧都要等到滤镜和编码结束，就会导致接收不及时。因此，程序分为3个独立的线程分别处理3个环节，前两个环节将处理的结果保存在队列中，继续处理，下一环节上一环节的队列中取数据进行处理。通过将各个环节拆分到不同的线程，避免了相互之间的性能影响。

为了简化代码，要求输入必须是vp8格式320*180格式的视频流。

# 运行
shell命令

启动一个输入视频流，注意必须是vp8，320x240（可使用asset目录中的文件）
```
ffmpeg -re -stream_loop -1 -i vp8-320x240.webm -an -c:v copy -f rtp -payload_type 100 rtp://127.0.0.1:5024
```

启动程序，需要指定输入，输出和水印文件
```
./rtwm.o input.sdp rtp://127.0.0.1:5034 watermark.png
```

输出通过monitor显示了各个环节的执行时间
```
timer=open_input elapse=10256939(ms) / 10(s) times=1
timer=open_output elapse=3455(ms) / 0(s) times=1
timer=decode elapse=139412199(ms) / 139(s) times=366
timer=read_frame elapse=138918873(ms) / 138(s) times=366
timer=filter elapse=205528(ms) / 0(s) times=355
timer=encode elapse=11660259(ms) / 11(s) times=355
timer=send_frame elapse=11387272(ms) / 11(s) times=355
timer=receive_packet elapse=952(ms) / 0(s) times=710
timer=write_frame elapse=239639(ms) / 0(s) times=355
timer=usleep elapse=154749807(ms) / 154(s) times=5849
```

# 关键代码
## 接收SDP文件作为输入
```
pFmtCtxIn = avformat_alloc_context();
pFmtCtxIn->iformat = av_find_input_format("sdp");
av_opt_set(pFmtCtxIn, "protocol_whitelist", "file,udp,rtp", 0);
avformat_open_input(&pFmtCtxIn, filename, 0, 0);
```
必须设置protocol_whitelist属性为"file,udp,rtp"才能用sdp文件作为输入。另外，应该用av_opt_set设置属性，而不是直接用等于，否则释放资源时会报指针错误。

## 从输入流读帧超时
```
if ((ret = av_read_frame(pFmtCtxIn, &packet)) < 0)
{
    if (AVERROR(ETIMEDOUT) == ret)
    {
        av_log(NULL, AV_LOG_WARNING, "input2decode thread av_read_frame timeout.\n");
        continue;
    }
    break;
}
```
如果程序启动时，还没有产生输入流，av_read_frame函数读不到内容就会超时，超时的时间是10秒（通过monitor可以观察到）。分析ffmpeg代码，发现最终调用了rtsp.c文件中udp_read_packet，该方法中每100毫秒取一次数据，共尝试100次，所以10秒内没有获得数据就会返回超时。

## 设置输出流的deadline属性
```
//realtime|good|best
av_opt_set(pCodecCtxOut->priv_data, "deadline", "realtime", 0);
```
为设置情况下，输出流有非常大的延时，几乎不可用，通过研究ffmpeg代码，发现可以通过设置deadline属性优化该问题。

## 设置输出流的payload_type属性
```
av_opt_set_int(pFmtCtxOut->priv_data, "payload_type", 100, 0);
```
为了便于和输出端对接，需要指定输出视频流中payload_type的值，如果不指定默认是96。指定的值和output.sdp文件中的值一致。

# 存在的问题
输出视频流的质量不好，经常出现花屏。

存在明显的延时。通过启动一个WebRTC客户端作为输入，加水印后输出到VLC浏览器，存在明显的延时。

忽略了音频处理，实际应用中必须解决音视频同步问题。