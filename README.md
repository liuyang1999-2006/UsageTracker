# 时长统计（UsageTracker）

一个统计手机各应用屏幕使用时长的安卓 App。基于系统提供的 `UsageStatsManager`，
不需要常驻后台服务，也不会上传任何数据——所有统计只在本地读取和展示。

> This is a small app that can track app usage time on an Android phone. Limited by my poor programming skills and my wishful way of communicating with AI, the app falls short in some minor details, but it can already perform the basic functions.
>
> 一个可统计安卓手机上 app 使用时间的小程序，受限于本人贫瘠的编程水平和许愿式的 AI 沟通方式，这个 app 在一些小细节是不合格，但是已经可以实现基础功能。

## 功能

- **五个时间维度**：今日 / 本周 / 本月 / 本年 / 本机（本周从周一开始；「本机」= 自手机启用以来至今——同时扫描按年与按月聚合桶的完整历史，取系统留存的最远时间作为起点，长区间自动改用聚合数据）
- **右上角日历**：点日历图标弹出 Material 日期选择器（可选范围限制在最早可统计日期到今天），选中某天即可查看当天的使用时长与**扇形统计图**；通过日期 chip 上的 × 退出日期视图回到 Tab 模式
- **扇形统计图**：当天使用占比环形图，前 8 名应用各占一色，其余合并为灰色「其他」，中心显示当天总计；列表色点与扇区颜色一一对应
- 汇总卡片：时间段总时长、使用过的应用数量
- 应用排行榜：图标、名称、包名、使用时长、占总时长的百分比、相对第一名的进度条
- 下拉刷新；授权页返回后自动刷新
- 不足 1 秒的记录视为误触，不计入统计

## 界面预览

- 首次启动展示授权引导页，点击按钮跳转系统「使用情况访问权限」设置
- 授权返回后自动加载列表

## 如何构建运行

### 环境要求

- Android Studio（Koala 及以上版本均可）
- JDK 17（Android Studio 自带，命令行构建需自行安装）
- Android SDK Platform 34（Android Studio 会自动下载）

### 方式一：Android Studio（推荐）

1. 打开 Android Studio → **Open** → 选择 `UsageTracker` 目录
2. 等待 Gradle Sync 完成（首次会下载 Gradle 8.7 与依赖，已配置腾讯云/阿里云镜像加速）
3. 连接手机（开启开发者选项和 USB 调试）或启动模拟器
4. 点击 **Run** 安装运行

### 方式二：命令行

```bash
cd UsageTracker
gradlew.bat assembleDebug      # Windows
# ./gradlew assembleDebug      # macOS / Linux
# 产物位于 app\build\outputs\apk\debug\app-debug.apk
adb install app\build\outputs\apk\debug\app-debug.apk
```

## 首次使用：授权

「使用情况访问权限」是系统特殊权限，无法弹窗授予，必须手动开启：

1. 启动 App，点击 **去开启权限**
2. 在系统设置列表中找到 **时长统计**，打开开关
3. 返回 App，自动开始统计

不同厂商 ROM 中该设置项名称略有差异，常见叫法：
「使用情况访问权限」「有权查看使用情况的应用」「使用情况访问」。

## 技术实现

| 模块 | 说明 |
| --- | --- |
| `UsageStatsHelper.kt` | 核心统计：短周期用 `queryEvents()` 事件流精确累加 `ACTIVITY_RESUMED → ACTIVITY_PAUSED` 间隔；本年/本机长区间改用 `queryAndAggregateUsageStats` 聚合数据（留存更久）；「本机」起点通过按年+按月聚合桶全历史扫描探测，取系统留存的最早时间 |
| `PieChartView.kt` | 自定义环形扇形图控件（Canvas 绘制），中心叠加总计时长；`PiePalette` 定义前 8 名配色，列表色点复用同一套颜色 |
| `PermissionUtils.kt` | 用 `AppOpsManager` 检查「使用情况访问权限」是否真正开启 |
| `MainActivity.kt` | 权限引导、Tab 切换、日历入口（`MaterialDatePicker`，选区范围约束到最早可统计日期）、日期模式与 Tab 模式切换、协程异步加载、下拉刷新 |
| `AppUsageAdapter.kt` | 排行榜列表（ViewBinding + RecyclerView），色点对应扇形图扇区颜色 |

几个关键点：

- **为什么用事件流而不是 `queryAndAggregateUsageStats`？**
  聚合接口按时间桶采样，「今日」粒度误差较大；事件流可以精确到每次前后台切换。
- **只显示有启动器入口的应用**（`getLaunchIntentForPackage != null`），
  过滤掉系统 UI 等无意义的记录；同时在 Manifest 中通过 `<queries>` 声明
  `MAIN/LAUNCHER` 可见性，满足 Android 11+ 的包可见性要求。
- 某些机型可能漏发 `ACTIVITY_PAUSED`，统计逻辑在遇到新的
  `ACTIVITY_RESUMED` 时会先结算上一个未闭合区间，避免丢算。

## 目录结构

```
UsageTracker/
├── app/src/main/
│   ├── java/com/usagetracker/app/
│   │   ├── MainActivity.kt        # 主界面：权限引导 + 数据展示
│   │   ├── UsageStatsHelper.kt    # 使用时长统计核心逻辑
│   │   ├── PermissionUtils.kt     # 特殊权限检查
│   │   └── AppUsageAdapter.kt     # 排行榜列表适配器
│   ├── res/                       # 布局、矢量图标、颜色、主题、文案
│   └── AndroidManifest.xml        # 声明权限与包可见性
├── build.gradle.kts / settings.gradle.kts
└── gradlew / gradlew.bat          # 命令行构建入口
```

## 已知局限

- 统计口径是「应用处于前台」，与系统自带的「屏幕使用时间」可能略有出入
  （例如息屏时是否立刻产生后台事件因 ROM 而异）。
- 各厂商 ROM 对使用统计服务的限制程度不同，个别 ROM 需允许 App 自启动
  才能稳定获取数据。
- App 图标为代码内置矢量图，如需上架可自行替换成设计资源。

## 版本信息

- v1.2（versionCode 3）：「本机」覆盖自手机启用以来的全部留存数据（按年+按月聚合桶全历史扫描探测起点），标题显示「自 …起至今」
- v1.1（versionCode 2）：新增右上角日历按日期查看 + 当天扇形统计图；新增本年 / 本机维度
- v1.0（versionCode 1）：今日 / 本周 / 本月 + 排行榜
- minSdk 24（Android 7.0）/ targetSdk 34（Android 14）
- Kotlin 1.9.24 · AGP 8.5.2 · Gradle 8.7 · Material Components 1.12
