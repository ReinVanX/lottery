# 音效文件目录

请将以下 3 个 MP3 文件放入此文件夹：

| 文件名 | 用途 | 播放方式 | 建议 |
|--------|------|---------|------|
| `draw.mp3` | 抽奖滚动中 | 循环播放 | ~3-8 秒，紧张悬疑感，如鼓点、心跳、轮盘旋转声 |
| `bingo.mp3` | 中奖揭晓瞬间 | 单次播放 | ~1-3 秒，短促有力，如"叮！"、锣声、提示音 |
| `win.mp3` | 中奖展示 | 循环播放 | ~3-8 秒，辉煌胜利感，如号角、烟花、欢呼声 |

## 播放流程

```
开始抽奖 → draw.mp3 循环播放
    ↓
揭晓瞬间 → draw.mp3 停止 → bingo.mp3 单次播放
    ↓
中奖画面 → win.mp3 循环播放
    ↓
确认/作废 → win.mp3 停止
```

## 音效来源建议

- **免费音效网站**：freesound.org、mixkit.co、pixabay.com/sound-effects
- **搜索关键词**：
  - draw: "suspense" / "tension" / "drum roll" / "spinning" / "countdown"
  - bingo: "ding" / "bell" / "chime" / "correct answer" / "notification"
  - win: "fanfare" / "victory" / "achievement" / "celebration" / "congratulations"

## 注意事项

- 文件必须是 `.mp3` 格式
- 文件名必须与上表一致
- 如果没有放置某个文件，程序会自动回退到内置合成音效
- `draw.mp3` 和 `win.mp3` 会自动无缝循环
- 文件放入后刷新页面即可生效
