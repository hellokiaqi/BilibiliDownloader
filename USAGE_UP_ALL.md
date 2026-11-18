# 下载指定 UP 主的全部视频

本文以 UP 主 `39597640` 为例，介绍本仓库提供的三种批量下载方式及其适用场景。日常在图形界面中挑选或全量备份某位 UP 主的公开投稿，优先使用**方式一**。

## 1. 准备工作

- **Java 环境**：已安装 JRE 8 或更高版本。Windows 用户直接双击 `release/Double-Click-to-Run-for-Win.bat`；macOS / Linux 用户在 `release/` 下运行 `java -Dfile.encoding=utf-8 -Dhttps.protocols=TLSv1.2 -jar INeedBiliAV.jar`。
- **FFmpeg 环境**：高画质 DASH 音视频需要 `ffmpeg` 混流合并为 `.mp4`。Windows 已内置 `release/ffmpeg.exe`；其他平台请确保系统 `PATH` 或 `bilibili.ffmpegPath` 中可调用 `ffmpeg`。
- **建议先登录账号**：未登录状态下 B 站投稿列表接口更容易触发 `-352` / `HTTP 412` 风控，且高码率/会员画质需要账号权限。点击主页左上角头像即可扫码登录。

---

## 2. 方式一：「UP 全部视频」列表页（推荐）

该方式对应 `TabUpAll` 与 `UpAllListFetcher`，打开标签页时只拉取投稿目录元数据，不为每条视频提前请求分 P 详情，加载上千条投稿依然流畅，仅在点击「下载所选」后才逐个解析勾选项。

### 操作步骤

1. 在 **主页** 的「链接或 ID」输入框中粘贴 `https://space.bilibili.com/39597640`，或直接输入纯数字 `39597640`（支持 3 位及以上的纯数字 `mid` 或任意包含 `space.bilibili.com/<mid>` 的链接）。
2. 点击右下角的 **「UP 全部视频」** 按钮（注意不是「查找」）。

![主页入口](release/preview/index.png)

3. 程序会新建并切换到 **「UP 39597640」** 标签页，自动分页拉取该 UP 的公开投稿（每页 `50` 条，安全上限 `200` 页即 `10,000` 条），底部状态栏实时显示 `正在加载第 X 页... 已获取 Y/Z 条`。

![UP 视频列表页](release/preview/upLibrary.png)

4. 列表加载完成后，可通过顶部工具栏筛选与勾选：
   - **年份**：下拉选择 `全部年份`、具体年份或 `年份未知`。已勾选的 BV 号独立保存，切换年份筛选时**不会丢失**其他年份的勾选状态。
   - **排序**：表格默认按 `发布日期` 倒序排列；点击 `标题`、`发布日期`、`BV号` 表头可切换升降序（`时长` 和 `播放量` 为格式化文本，不参与排序）。
   - **全选 / 清空**：仅对**当前筛选条件下可见的行**生效。例如切到 `2024` 年后点「全选」，只会勾选 2024 年的全部投稿。
   - **清晰度**：默认读取 `bilibili.menu.download.qn`（默认 `1080P`），也可手动改选 `8K`、`4K`、`1080P60`、`CC字幕`、`弹幕` 等。
5. 点击 **「下载所选」**：
   - 程序会按 `max(200ms, bilibili.download.period.between.query)` 的间隔依次解析所选 BV 的全部分 P 并送入下载队列；
   - 同一面板中已入队过的 BV 会自动跳过，避免重复排队；
   - 全部入队后自动切换到 **「下载页」**，可在下载页执行 `全部继续`、`全部暂停`、`全部删除`、`删除已完成`，或对单个任务执行 `暂停`、`打开文件`、`打开文件夹`、`删除任务`。

![下载页](release/preview/download.png)

### 接口限流与「继续加载」断点续拉

B 站空间投稿接口（`x/space/wbi/arc/search`）对连续翻页非常敏感。`UpAllListFetcher` 内置了以下保护逻辑：

- **翻页限速**：每页之间至少间隔 `300 ms`；若 `config/app.config` 中的 `bilibili.download.period.between.query` 大于 `300`，则按配置值等待。
- **自动退避与旧接口回退**：单页遇到风控错误码（`-799`、`-352`、`-412`、`-509`、`-500`、`-10403`）或网络/`HTTP 412` 异常时，先按 `1s`、`2s`、`3s` 退避重试最多 3 次；仍失败则回退到非 WBI 接口 `x/space/arc/search` 再尝试 1 次。
- **保留进度与断点续拉**：如果重试后仍被拦截，**已经拉取到的投稿会完整保留在表格里**，底部状态栏会显示具体原因（如 `触发 B 站风控(412)，请稍后点「继续加载」或登录后再试`），同时顶部 **「继续加载」** 按钮变为可用。稍等几分钟（或登录、点击 `操作 → 登录相关 → 刷新Cookie`）后点 **「继续加载」**，即可直接从中断的那一页继续往后拉取并按 BV 号自动去重合并；点 **「刷新」** 则会从第 1 页重新拉取。

---

## 3. 方式二：`?full=1` 参数分页解析

如果想在传统「视频」标签页里按关键词或播放量/收藏量排序查看某几页投稿，可在空间链接后加上 `full=1`（或 `full=true`、`full=all`）：

```text
https://space.bilibili.com/39597640/video?full=1
```

也可以组合分区 `tid`、排序 `order`（`pubdate` 最新发布、`click` 最多播放、`stow` 最多收藏）与关键词 `keyword`：

```text
https://space.bilibili.com/39597640/video?tid=0&order=click&keyword=教程&full=1
```

粘贴到主页后点击 **「查找」**（翻页点主页或标签页右上角的 **「下一页」**）：
- 该方式会命中 `URL4UPAllFullParser`（权重高于默认的 `URL4UPAllMedialistParser`），单页展示条数由主页「每页数量」（`bilibili.pageSize`）决定，底层接口单页上限为 `50` 条。
- **注意**：与方式一不同，`URL4UPAllFullParser` 在打开页面时就会为当前页的每条 BV 额外请求一次视频详情以展开分 P，网络请求多、耗时较长，大批量下载不建议使用此方式。

---

## 4. 方式三：配置文件一键下载与周期轮询（`batchDownload.config`）

适合“定期追更某几个 UP 主或收藏夹，自动跳过已下载视频”的无人值守场景。

### 配置文件位置

程序在**当前数据目录**下的 `config/` 文件夹中读取配置：
- 直接运行 `release/` 目录下的脚本或 `INeedBiliAV.jar` 时，文件路径为：`release/config/batchDownload.config`
- 若通过 JVM 参数 `-Dbilibili.prop.dataDirPath=<目录>` 指定了数据目录，则路径为：`<目录>/config/batchDownload.config`
- 若在 IDE 或仓库根目录直接运行未打包类，则路径为：`config/batchDownload.config`

`config/` 下所有文件名匹配正则 `^batchDownload.*\.config$` 的文件都会出现在菜单 **配置 → 一键下载配置** 中供切换。

### 配置示例

```ini
[url:https://space.bilibili.com/39597640/video]
start.page = 1
stop.condition = page:10
stop.bv.bounds = exclude
stop.alert = true
download.condition = _!downloaded
```

### 字段与条件语法

- **目标区块头**：
  - `[url:<链接>]`：任意支持分页或单页解析的链接（如 UP 空间视频页、合集、收藏夹链接）。
  - `[favorite:all]` 或 `[favorite:默认收藏夹,稍后再学]`：需先登录账号，自动展开为对应收藏夹任务。
- **任务参数**：
  - `start.page = 1`：起始页码（默认 `1`）。
  - `stop.bv.bounds = exclude`：命中 `stop.condition` 的那条边界视频本身是否下载，`include` 为下载，`exclude` 为不下载（默认 `exclude`）。
  - `stop.alert = true`：单个区块完成后是否弹窗提示（默认 `true`；按计划周期下载时仅写日志不弹窗）。
- **`download.condition` 与 `stop.condition` 表达式**：
  - `_:_`：无条件匹配所有条目。
  - `_!downloaded`：仅匹配尚未记录在 `config/repo.config` 中的条目（按当前菜单优先清晰度判断）。
  - `page:10` / `page>5` / `page<3`：按页码判断（注意：`page>5` 在源码中含义为 `5 > 当前页`，即前 4 页；要写“超过第 10 页停止”请写 `stop.condition = page:11` 或 `page<10` 表示 `10 < 当前页`）。
  - `bv:BV1BJ411E7uM`：匹配指定 BV 号（常用于 `stop.condition` 追更到上次最新一条即停）。
  - `cTime>2024-01-01` / `cTime<2025-01-01`：按投稿发布日期判断（需解析器提供 `cTime` 字段）。
  - `favTime>2024-01-01`：按收藏日期判断（仅收藏夹解析有效）。
  - `avTitle:.*合集.*` / `avTitle!.*预告.*`：按视频主标题匹配/排除 Java 正则表达式。
  - `clipTitle:.*上篇.*`：按分 P 小标题匹配/排除 Java 正则表达式。
  - **组合规则**：同一行内用英文逗号 `,` 分隔表示**同时满足（AND）**；多行 `download.condition` 或多行 `stop.condition` 之间表示**满足其一（OR）**。

### 启动执行

1. 菜单栏 **配置 → 优先清晰度** 确认目标清晰度（默认 `1080P`），**配置 → 一键下载配置** 确认配置文件名。
2. 单次运行：点击 **操作 → 一键下载**（翻页间隔固定包含 `1500 ms` 休眠）。
3. 周期运行：点击 **操作 → 按计划周期下载**，或设置 `bilibili.download.batch.plan.runOnStartup = true` 随程序启动自动运行（轮询时间段由 `bilibili.download.batch.plan` 控制，支持 `bilibili.download.push.type = Mail` 邮件推送汇总）。

---

## 5. 保存路径与清晰度降级说明

- **默认保存目录**：相对数据目录的 `download/`（例如 `release/download/`），可在 `config/app.config` 中修改 `bilibili.savePath`。
- **文件防覆盖与历史去重**：默认开启 `bilibili.repo = on`（已记录在 `config/repo.config` 中的视频不会重复下载）和 `bilibili.name.autoNumber = true`（同名文件自动加 `(01)` 序号）。
- **清晰度自动降级**：`bilibili.download.quality.autoFallback` 默认为 `true`。当所选清晰度（如 `4K` 或 `1080P+`）高于该视频实际可用清晰度或当前账号无权限时，程序会自动下载实际可用的最高清晰度并在日志中打印说明；若希望严格报错拦截，可将该项设为 `false`。

---

## 6. 常见问题速查

| 现象 | 原因与处理方式 |
| --- | --- |
| 状态栏提示 `-799` / `-352` / `触发 B 站风控(412)` | 接口限流。先登录账号，或点 `操作 → 登录相关 → 刷新Cookie`，在 `config/app.config` 将 `bilibili.download.period.between.query` 设为 `800`～`1500`，稍等后点 **「继续加载」**。 |
| 点击「UP 全部视频」提示“请输入 UP 主页链接或 mid 数字 ID” | 输入框内容既不包含 `space.bilibili.com/<数字>` 也不是 3 位以上纯数字（例如误填了 `BV` 号）。 |
| 个别勾选的视频入队失败 | 状态栏会显示失败条数，常见原因为视频已失效、充电专属/会员受限，或查询过快触发风控。 |
| 下载后的视频只有画面没声音，或停留在 `.m4s` | 未检测到可用的 `ffmpeg`。确认 `release/ffmpeg.exe` 存在，或在 `config/app.config` 正确配置 `bilibili.ffmpegPath`。 |
| 双击启动脚本一闪而过 | Windows 下改用 `release/Double-Click-to-Run-for-Win-debug.bat` 运行，根据控制台保留的报错信息排查 Java 环境或目录权限。 |
