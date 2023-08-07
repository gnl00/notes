# ffmpeg



## 转换

mp4 to flv 带关键帧

```bash
ffmpeg -i input.mp4 -vcodec copy -acodec copy -flvflags add_keyframe_index output.flv
```



## 推/拉流

推流

```bash
# -re 表示实时模式读取输入，按照实际的帧速率进行处理
ffmpeg -re -i output.flv -c copy -f flv rtmp://localhost/live/livestream
```

指定时长推流推流

```bash
# 推流 10 分钟
ffmpeg -re -i input.flv -c copy -f flv -t 00:10:00 rtmp://localhost/live/livestream
```

循环推送

```bash
# 加上参数 -stream_loop -1 表示循环推送
ffmpeg -re -stream_loop -1 -i input.flv -c copy -f flv rtmp://localhost/live/livestream
```

拉流

```bash
ffplay rtmp://localhost/live/livestream
```



## 参考

* https://blog.csdn.net/guoyunfei123/article/details/106189162
