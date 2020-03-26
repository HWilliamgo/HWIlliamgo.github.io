用<url>来表示视频文件的地址。

---

查看关键帧数据：
``` bash
ffprobe -show_frames -select_streams v:0 -print_format csv <url> | grep ",I,"
```

查看流信息：
``` bash
ffprobe -v quiet -print_format json -show_format -show_streams <url>
```

查看分辨率：
``` bash
ffprobe -v debug -select_streams v -skip_frame nokey -count_frames <url> 2>&1|grep Reinit
```

查看每一帧的音频数据：
``` bash
ffprobe -show_frames -select_streams a -print_format csv <url>
```

提取视频的h264码流
``` bash
ffmpeg -i <url> -codec copy -f h264 video.h264
```
