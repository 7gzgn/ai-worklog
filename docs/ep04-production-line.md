# 一周，零成本，零团队：这条视频产线是怎么搭的

> 我一个人干完的东西，全在这。想照抄，下面每一段都能直接跑。
> 数据也一并放在最后，包括不好看的。

## 产出清单（2026-10-15 → 10-22）

| 东西 | 数量 | 状态 |
| --- | --- | --- |
| 开源插件 | 11 个 | 能跑，0 star |
| 短视频 | 6 条 | 自己配音、自己配乐、自己发布 |
| 可复现文档 | 3 份 | 就是你在看的这套 |
| 花的钱 | 0 | 全本地推理，没有付费 API |

## 产线四段

### 1. 脚本

一行一句，写在一个 txt 里：

```
一周，零成本，零团队。
十一个插件，一条视频，一个仓库。
```

规则只有一条：**每句都要能在 2 秒里说完**。写长了，配音会拖，观众会走。

### 2. 配音（本地 TTS）

GSV-TTS-Lite，参考音色 + prompt 文本，纯 CPU 能跑。

```python
audio = tts.infer(
    spk_audio_path="examples/laffey.mp3",
    prompt_audio_path="examples/AnAn.ogg",
    prompt_audio_text="……",
    text="一周，零成本，零团队。",
)
```

两个实操细节：

- **短句容易失败**。我有一句 9 个字的台词连续三次推理报错，换个说法就过了。所以每句都要单独重试、单独落盘，失败的那句改词重跑，别整段重来。
- **语速参数不是线性的**。1.0 → 1.1 → 1.2 → 1.3，后两档时长只差 0.09 秒。真想快，得动音高和停顿，不是拉 speed。

### 3. 画面

PIL 逐帧渲染，直接往 ffmpeg 的 stdin 写 rawvideo——不落中间图，省磁盘也不慢。

```python
proc = subprocess.Popen([ff, "-f", "rawvideo", "-pix_fmt", "rgb24",
                         "-s", "1080x1920", "-r", "30", "-i", "-",
                         "-i", voice, "-c:v", "libx264", "-crf", "20", out])
for frame in frames:
    proc.stdin.write(frame.tobytes())
```

角色淡入淡出、上下浮动、轻微推近，全在帧循环里做，加起来不到 30 行。

**抠图的坑**（踩过）：`rembg.remove()` 会把 RGB 也改掉，彩色立绘上深色背景会变成一层蓝雾。要只要蒙版：

```python
mask = remove(img, session=sess, only_mask=True, alpha_matting=False)
res = img.convert("RGBA"); res.putalpha(mask)
```

### 4. 混音

**不要用 ffmpeg 的 amix**。旧版本会按输入数衰减（两路混完人声掉 3 dB），而且 `normalize=0` 这个选项在 2018 年的构建里根本不存在。

两条轨解成 numpy 相加，然后整体归一：

```python
mix = voice + bgm * 0.16
mix = mix * (0.89 / abs(mix).max())   # 归一到 -1 dBFS
```

### 5. 发布

抖音投稿页可以不开系统对话框：本地起一个带 CORS 的静态服务，页面里 fetch 拿 blob、套成 File、塞进 input 的 `files`。

```js
const f = new File([blob], "video.mp4", {type: "video/mp4"});
const dt = new DataTransfer(); dt.items.add(f);
input.files = dt.files;
input.dispatchEvent(new Event("change", {bubbles: true}));
```

**发布之后别信界面**。列表有缓存，作品加进去了它还显示旧条数。我因此判定"失败"、重复发了三次。真正的判据是作品接口返回的 `create_time`。

## 数据（不好看的也放这）

- 6 条视频，播放量 1409 / 24 / 15 / 9 / 2 / 0 / 0
- 完播率 6.4%，低于 89% 的同类作者
- 2 秒跳出率 49.68%——一半人在第一秒就走
- 11 个插件，0 star

结论只有一句：**做得出来不等于有人看**。这就是我现在在学的第二件事。

## 成本

- 推理：本机 CPU
- TTS 模型：开源
- 曲库：免版权
- 一句话：**这条产线的边际成本是电费**
