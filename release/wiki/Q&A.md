# 常见问题与配置排障（Q&A）

这里汇总日常使用、自定义配置、接口风控、FFmpeg 混流与账号登录的常见问题。

## 核心配置文件一览

所有相对路径均相对于**程序数据目录**（运行 `release/` 下脚本或 JAR 时即 `release/` 目录；若传入了 JVM 参数 `-Dbilibili.prop.dataDirPath=<目录>`，则为该指定目录）：

- `config/app.config`：主配置文件（可通过菜单 **配置 → 打开配置页** 可视化修改并保存，或直接用文本编辑器以 UTF-8 编码编辑）。
- `config/cookies.config`：本地保存的登录 Cookie 与 `refresh_token`（切勿泄露给他人）。
- `config/repo.config`：已完成下载记录仓库（格式如 `BV1BJ411E7uM-80-p1`，每行一条，用于防重复下载）。
- `config/tasks.config`：通过菜单 **操作 → 保存下载任务 / 加载下载任务** 导出的队列快照。
- `config/batchDownload.config`：**操作 → 一键下载** 使用的规则配置文件。
- `config/qr-login.png`：扫码登录时自动生成的二维码兜底图片。

---

## 一、启动、权限与环境

<details>
<summary>双击启动脚本没反应，或者窗口一闪而过怎么办？</summary>

1. **Windows**：请双击 `release/Double-Click-to-Run-for-Win-debug.bat`，该脚本使用 `java` 而非 `javaw` 并在退出后暂停保留控制台输出，可直接看到具体报错（如未安装 Java 8+ 或 JAR 未完整解压）。
2. **macOS / Linux**：`Double-Click-to-Run-for-Mac.command` 默认启动 `launch.jar`。若直接克隆源码仓库运行且尚未编译 `launch.jar`，请在终端进入 `release/` 目录执行：
   ```bash
   java -Dfile.encoding=utf-8 -Dhttps.protocols=TLSv1.2 -jar INeedBiliAV.jar
   ```
</details>

<details>
<summary>启动时弹窗提示「检测到程序对于数据目录没有“写”权限」怎么办？</summary>

程序需要在数据目录下读写 `config/` 配置、Cookie、历史记录以及默认 `download/` 目录。常见于把程序放在了 `C:\Program Files` 等系统受保护目录。解决方式任选其一：
- **推荐**：将整个程序文件夹移到普通用户有读写权限的位置（如 `D:\Tools\BilibiliDownloader`）。
- **指定独立数据目录**：在启动命令中加入 JVM 参数 `-Dbilibili.prop.dataDirPath="D:\BilibiliData"`（目录需提前建好）。
</details>

<details>
<summary>部分系统或显卡驱动下主窗口空白、控件错位怎么办？</summary>

- 如果开启了跟随系统主题导致字体或布局异常，可在 `config/app.config` 中确保 `bilibili.theme = default`。
- 若使用 `launch.jar` 启动时遇到渲染兼容问题，可通过 JVM 参数切换旧版窗口入口类：
  ```bash
  java -Dfile.encoding=utf-8 -Dbilibili.prop.mainClass=nicelee.ui.FrameMain_v3_4 -jar launch.jar
  ```
</details>

---

## 二、FFmpeg 与音视频合并 / 格式

<details>
<summary>为什么必须配置 FFmpeg？提示合并失败或只留下两个 .m4s 文件怎么办？</summary>

B 站 Web 端高画质视频均采用 DASH 音视频分离流（`.m4s`），下载完成后必须调用 `ffmpeg` 将画面和音频无损混流为 `.mp4`。
- **查找顺序**：程序启动时先检查 `bilibili.ffmpegPath`（相对数据目录或绝对路径），再检查系统环境变量 `PATH` 中的 `ffmpeg`。
- **Windows**：仓库已自带 `release/ffmpeg.exe`，从 `release/` 启动可直接识别。
- **Linux / Windows 自动下载**：在 `win_amd64`、`win_arm64`、`linux_amd64`、`linux_arm64` 下若未检测到 `ffmpeg`，启动时会弹窗询问是否下载约 3MB 的精简编译版（如默认源下载慢，可在 **配置 → FFMPEG源选择** 切换 `Supabase` / `Github` / `Cloudinary` / `Twicpics`）。
- **macOS 或自定义安装**：执行 `brew install ffmpeg`，或在 `config/app.config` 中将 `bilibili.ffmpegPath` 设为 `ffmpeg` 可执行文件的完整路径（如 `/opt/homebrew/bin/ffmpeg`）。
- 如需查看 `ffmpeg` 报错详情，可在 `config/app.config` 中设 `bilibili.cmd.debug = true` 并用 debug 模式启动。
</details>

<details>
<summary>下载好的 MP4 用系统自带播放器打开只有声音没有画面（黑屏）？</summary>

`app.config` 默认的视频编码优先级为 `bilibili.dash.video.codec.priority = 12, 7, 13`，即优先选择体积更小的 `12`（HEVC / H.265），其次才是 `7`（AVC / H.264）和 `13`（AV1）。部分缺少 HEVC 解码器的老旧播放器会黑屏。
- 解决方法一：使用 PotPlayer、VLC、IINA、mpv 等支持 HEVC/AV1 的播放器。
- 解决方法二：在 `config/app.config` 中改为优先下载兼容性最好的 AVC（H.264）：
  ```ini
  bilibili.dash.video.codec.priority = 7, 12, 13
  ```
</details>

<details>
<summary>如何只下载音频，并保存为 .mp4 / .aac / .flac / .mp3？</summary>

- **临时切换**：右上角菜单 **配置 → 下载模式(DASH) → 仅音频**（重启后恢复配置文件设置）。
- **持久配置**：在 `config/app.config` 中修改：
  ```ini
  # 0: 下载音视频并合并  1: 仅下载视频  2: 仅下载音频
  bilibili.dash.download.mode = 2
  bilibili.dash.suffix4AudioOnly = .mp4
  ```
- **格式说明**：
  - `.mp4`（默认）或 `.aac`：使用默认命令 `-c:a copy` 直接复制音频流，无需重编码，速度最快（Hi-Res 无损音轨请将后缀改为 `.flac`）。
  - `.mp3`：需要重编码，**仓库自带的精简版 `ffmpeg.exe` 不包含 `libmp3lame` 编码器**。若需转 `.mp3`，请自行安装完整版 FFmpeg，并同时修改转换命令：
    ```ini
    bilibili.dash.suffix4AudioOnly = .mp3
    bilibili.dash.ffmpeg.command.transAudioOnly = {FFmpeg}, -y, -i, {SavePath}{AudioName}, -vn, -acodec, libmp3lame, -q:a, 2, {SavePath}{DstName}
    ```
</details>

<details>
<summary>为什么配置了 MP4（bilibili.format = 0），极少数老视频下载出来仍是 FLV？</summary>

`bilibili.format = 0` 表示**优先**请求 DASH（HTML5）流；少数远古视频或特殊剧集在 B 站服务端仅有分段 FLV 源，程序会自动回退到 FLV 下载并用内置 `FlvMerger`（或 `bilibili.flv.ffmpeg = true` 调用 FFmpeg）合并。
</details>

---

## 三、登录、清晰度与风控限流

<details>
<summary>为什么实际下载下来的清晰度比选择的低（例如选了 4K / 1080P60 却下了 1080P 或 480P）？</summary>

1. **账号权限与登录状态**：未登录时高码率流受限；`4K`、`1080P60`、`1080P+`、`HDR`、`杜比/Hi-Res` 等通常需要登录大会员账号，或源视频本身最高只上传了较低分辨率。
2. **自动降级开关**：本分支默认开启 `bilibili.download.quality.autoFallback = true`，当期望清晰度不可用时会自动下载接口实际返回的最高画质并在日志中打印提示。若你希望画质低于预期（低于 720P）时直接弹窗报错中止，可设：
   ```ini
   bilibili.download.quality.autoFallback = false
   bilibili.alert.qualityUnexpected = true
   ```
</details>

<details>
<summary>扫码、密码、短信三种登录方式有什么区别？遇到验证码打不开怎么办？</summary>

- **二维码登录**（`bilibili.user.login = qr`，推荐）：无需过极验验证码，点击主页头像或 **操作 → 登录相关 → 二维码登录** 即可。若部分桌面环境下扫码小窗口被遮挡，可直接打开 `config/qr-login.png` 扫码。
- **用户名密码 / 短信验证登录**（`pwd` / `sms`）：需要人工完成极验滑块/点选验证。程序会在本地监听 `127.0.0.1:8787` 并自动调用默认浏览器打开验证页：
  - 若端口 `8787` 被其他程序占用，请修改 `bilibili.server.port` 后重试；
  - 短信登录的手机号输入框必须包含 `+国际区号` 和**空格**，例如 `+86 18812344321`。
</details>

<details>
<summary>刷 UP 主视频列表或批量解析时提示「触发 B 站风控(412) / -352 / -799 请求过于频繁」怎么办？</summary>

1. **先登录账号**，并点击菜单 **操作 → 登录相关 → 刷新Cookie**（该操作会向 B 站接口上传浏览器指纹参数，有助于缓解 `-352` 风控）。
2. 在 **「UP 全部视频」** 标签页遇到中断时，无需关闭重来，等 1～3 分钟后直接点 **「继续加载」** 即可从断点页续拉。
3. 在 `config/app.config` 中调大查询间隔（单位毫秒），降低连续请求频率：
   ```ini
   bilibili.download.period.between.query = 1000
   ```
4. 请勿随意单独修改 `bilibili.userAgent.pc`；若确需修改，需按 `src/resources/app.config` 注释说明同步抓包更新 `bilibili.userAgent.pc.fingerprint` 与 `bilibili.userAgent.pc.payload`。
</details>

---

## 四、下载目录、自定义命名与去重仓库

<details>
<summary>怎样修改下载保存目录和自定义文件命名格式？</summary>

修改 `config/app.config`（或在 **配置 → 打开配置页** 中通过右侧 `...` 文件夹选择器修改）：

```ini
# 支持相对路径或绝对路径
bilibili.savePath = D:/Videos/Bilibili/

# 常用命名变量：
# avId(BV号) | numAvId(数字av号) | pAv / pAv2(分P号，数字代表补零宽度)
# pDisplay / pDisplay3(列表序号) | qn(清晰度代码，如80代表1080P)
# avTitle(主标题) | clipTitle(分P标题) | UpName(UP主名) | UpId(UP主mid)
# listName(合集/收藏夹名) | listOwnerName | listOwnerId | favTime | cTime
# 条件语法：(:变量 存在时输出内容)  (!变量 不存在时输出内容)  也可用方括号 [:变量 ...]
bilibili.name.format = (:listName 0_listName/)UpName/avTitle-(:cTime cTime-)pAv2-qn
```

- 如果单 P 视频的 `avTitle` 与 `clipTitle` 完全一样，不想在文件名里重复两遍，可开启 `bilibili.name.format.clipTitle.allowNull = true`，并配合 `(:clipTitle -clipTitle)` 使用。
- 请确保 `bilibili.name.doAfterComplete = true`（默认开启），下载完成后会自动重命名；若设为 `false`，重命名指令只会追加到下载目录的 `rename.bat` 中等待手动执行。
</details>

<details>
<summary>提示「您已经下载过视频」无法再次下载，或者批量下载弹窗太多怎么办？</summary>

程序默认开启仓库去重（`bilibili.repo = on`），成功下载的视频会写入 `config/repo.config`：
- **临时允许重复下载**：右上角菜单 **配置 → 下载前先查询记录? → 不查询**（重启后恢复）。
- **同一视频想再下载另一个清晰度**：在 `config/app.config` 中设 `bilibili.repo.definitionStrictMode = on`（默认 `off` 表示不区分清晰度，下过任意画质即视为已下载）。
- **永久关闭去重检查**：设 `bilibili.repo = off`（若连记录也不想写入，同时设 `bilibili.repo.save = off` 并删除 `config/repo.config`）。
- **减少批量下载时的已下载弹窗**：设 `bilibili.alert.isAlertIfDownloded = false`，或调整上限 `bilibili.alert.maxAlertPrompt = 5`。
</details>

---

## 五、下载速度、断点续传与网络代理

<details>
<summary>视频没下完意外关掉了程序，重新下载会从头开始吗？</summary>

不会。未完成的任务在下载目录保留有 `.part` 临时文件（注意：删除任务卡片会一并清理对应的 `.part` 文件）。
- 退出前可通过菜单 **操作 → 保存下载任务** 将当前下载页队列保存到 `config/tasks.config`，下次启动后点 **操作 → 加载下载任务** 即可恢复并从断点续传。
- 若关闭了 `bilibili.restrictTempMode = off` 排查异常残留文件，可在正常后改回 `on`，程序关闭时会自动清理已完成视频遗留的无效临时文件。
</details>

<details>
<summary>下载速度很慢，或者任务排队很久后报 403 失败怎么办？</summary>

1. **绕过慢速 PCDN 节点**：当分配到 `*.mcdn.bilivideo.cn` 等节点速度很慢时，可点击菜单 **配置 → 音视频链接替换host? → 替换**（或持久化设置 `bilibili.download.host.forceReplace = true`），程序会将匹配 `upgcxcode` 的链接替换为镜像节点（`bilibili.download.host.alternative`）；也可尝试 `bilibili.download.forceHttp = true`。*注：节点替换视网络运营商情况可能变快也可能报 403，如异常请关回 `false`。*
2. **链接超时自动重查**：一次向队列提交成百上千个任务时，B 站下载链接一段时间后会失效。程序默认在等待超过 `bilibili.download.urlValidPeriod = 90`（分钟）时自动重新查询下载链接；对于手动点重试的任务，可在菜单 **配置 → 下载重试策略 → 重新查询url**（或设 `bilibili.download.retry.reloadDownloadUrl = true`）开启重查。
3. **并发任务数**：通过 `bilibili.download.poolSize = 3` 调整同时下载的视频个数，不建议设得过大以免触发连接限制。
</details>

<details>
<summary>如何配置 HTTP / HTTPS 或 SOCKS 代理？</summary>

在 `config/app.config` 中取消对应代理配置的注释（**HTTP/HTTPS 代理与 SOCKS 代理二选一，不要同时开启**）：

```ini
# 方式一：HTTP + HTTPS 代理
proxyHost = 127.0.0.1
proxyPort = 7890

# 方式二：SOCKS 代理
#socksProxyHost = 127.0.0.1
#socksProxyPort = 1080
```
</details>
