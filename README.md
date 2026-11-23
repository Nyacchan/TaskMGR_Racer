# SSD_Racer

在 Windows 任务管理器「性能」页的那条曲线上开车。

任务管理器画什么，车就跑在什么上面。切到 CPU 页，它跑 CPU 利用率曲线；切到磁盘页，它跑磁盘活动曲线；内存、GPU 一样。曲线的每一个峰谷都是赛道上的坡，图像往左滚动，车被拖出画面就输了。

这个项目是从 [CS-LX/CPURacer](https://github.com/CS-LX/CPURacer) 二次开发来的，见文末「来源与授权」。

---

## 玩法

| 按键 | 作用 |
| --- | --- |
| `W` / `S`（或 ↑ / ↓） | 油门。`W` 前进，`S` 倒车。油门范围 -100% ~ +100%，0 是分界 |
| 空格 | 开始 / 重开 |
| `R` | 翻车或出界之后重来 |

- 油门松手会慢慢回中，不回中就不是车了。
- 车翻过来（车顶碰到曲线）、开出画面左右任一侧、掉出曲线下方，都算结束。
- 曲线的起伏就是地形。上坡要加油，下坡会自己滑，坡太陡冲不上去就得退回来重来过。
- 距离按走过的像素折算成米，最好成绩存在内存里（关掉就没了）。

**开之前**：任务管理器要开着，并且停在「性能」页的某个图表上。程序抓不到窗口就不会画任何东西，看着像坏了，其实是在等。

**权限**：本程序和任务管理器必须同权限。任务管理器是管理员启的，本程序也得用管理员启，否则抓不到画面（Windows 的会话隔离）、键盘也按不进去（UIPI）。

---

## 原理

整个程序就五步，一步都省不了。

### 1. 找到那张图

任务管理器是 Win11 的 XAML 应用，主窗口类名 `TaskManagerWindow`，里面每个图表都是一个 `CvChartWindow` 子窗口。侧栏那排缩略图也是 `CvChartWindow`，所以不能拿第一个，得遍历所有子窗口，挑面积最大、宽 ≥ 200、高 ≥ 150 的那个当主图。

跟随用两路：`SetWinEventHook` 监听 `EVENT_OBJECT_LOCATIONCHANGE`（窗口动了、缩放了）和 `EVENT_SYSTEM_FOREGROUND`（切前台了），再加一个 120ms 的定时重扫兜底。拖动窗口时任务管理器偶尔会短暂枚举不到图表，这时候如果上一次的图表句柄还可见，就吞掉这次「没找到」，不然覆盖层会一闪一闪。

原生这层在 `SSDRacer.TrackNative`（C++ DLL），找到的图表矩形换算成物理像素交给上层。

### 2. 把像素取出来

**关键坑：不能 `BitBlt` 图表的窗口 DC。** 那条彩色曲线是合成层画上去的，窗口自己的 DC 里只有灰白网格，拿不到线。

正确做法是 `Windows.Graphics.Capture` 对任务管理器窗口开会话，拿合成后的帧，再按 `DWMWA_EXTENDED_FRAME_BOUNDS` 的屏幕坐标裁出图表区域。开会话前把 `IsBorderRequired` 设成 false，不然截出来的画面带一圈黄色边框。

覆盖层是独立窗口，不进这个截图的帧，所以不会出现「车被自己拍进去当赛道」的自我反馈。

### 3. 把曲线变成高度场

拿到图之后，逐列找曲线在哪。不是找填充色带，是找那条线的「脊」：

- **动态学主题色**。系统主题换了、图表配色换了，硬编码一个蓝色就不灵了。先按饱和度和亮度自举出一批候选，之后要求主通道一致、通道差比例达标。
- **每列取最优点**。一列里可能有多个符合颜色的像素（线有粗细、抗锯齿、还有网格线），取分数最高的，再在附近 ±3 像素内按 85% 峰值加权求质心，得到亚像素级的脊线位置。
- **空间连续性**。以上一列的结果为中心 ±32 像素搜，脱靶才整列重扫。曲线是连续的，这个先验比逐列独立找稳得多。
- **后处理**。缺列线性插值，孤立尖刺（14 像素以内、左右都是平的）修掉，最后半径 1 平滑一次。不能过度平滑，真实的山谷得留着，那是赛道。

有效列覆盖不到 25% 的时候不拒绝整帧，缺的列按图底补一条平地。否则旧的悬空地形会把车卡在半空。

这套东西在 `SSDRacer.Capture`。

### 4. 车是真刚体

物理用 Box2D（`Box2D.NetStandard` 1.0.4）。车是车身方块加两个圆轮，轮子用 revolute joint 挂在车身上，电机直接给转速和扭矩。

车活在**世界坐标**里，不是每帧被搬到曲线上贴住。图每帧往左滚，程序拿前后两帧的高度场做配准，估出这一帧滚了几个像素，再把这几个像素加进滚轴原点，只把右边缘新出现的列接进世界折线。所以车是被滚动的赛道带走的，油门和惯性都是真的。

一帧最多认 24 像素的滚动量，平滑系数 0.25。估多了会跳，估少了会漂。

### 5. 画上去

覆盖层是个分层窗口：`WS_EX_LAYERED | WS_EX_TRANSPARENT | WS_EX_TOPMOST | WS_EX_NOACTIVATE | WS_EX_TOOLWINDOW`，用 Direct2D 画车和 HUD。

**不画赛道**。赛道就是任务管理器自己画的那条线，再描一遍只会和原图打架、还不同步。

**不抢焦点**。`WS_EX_NOACTIVATE` 加上 `SW_SHOWNOACTIVATE`，点它、它浮在上面，都不会把任务管理器挤下去。键盘靠 `GetAsyncKeyState` 轮询，外加一个 `WH_KEYBOARD_LL` 镜像，因为 Win11 上任务管理器占着前台时 `GetAsyncKeyState` 偶尔不灵。

跨进程的相对 Z 序是拿不到的：`SetWindowPos` 指定插入位置、`CreateWindowEx` 指定 owner 都会被拒（`ERROR_ACCESS_DENIED`）。所以策略是「永远置顶 + 只在任务管理器或自己在前台时才显示」，别的前台就收起来。

---

## 目录结构

```
src/
  SSDRacer.App          托盘程序、主循环（60Hz）、热键
  SSDRacer.TrackNative  C++ DLL：找任务管理器窗口、跟图表、监听窗口事件
  SSDRacer.Taskmgr      图表句柄、ROI、坐标映射
  SSDRacer.Capture      Windows.Graphics.Capture 取帧 + 高度场提取
  SSDRacer.Game         Box2D 物理、卷轴配准、判负、金币
  SSDRacer.Overlay      Direct2D 分层覆盖层、HUD、车
  SSDRacer.Native       P/Invoke
  SSDRacer.Localization 中英文案
  SSDRacer.Tests        单元测试
```

技术栈：.NET 8（`net8.0-windows10.0.20348.0`）、WPF、C++（C++20）、Box2D.NetStandard 1.0.4、Vortice.Direct2D1 3.8.3。

---

## 编译

需要 .NET 8 SDK（装 10 也行）和带 C++ 桌面的 Visual Studio Build Tools。

```powershell
# 1. 先编 C++ 跟踪 DLL
MSBuild.exe SSDRacer.TrackNative/SSDRacer.TrackNative.vcxproj `
    /m /p:Configuration=Release /p:Platform=x64

# 2. 再编主程序
dotnet build SSDRacer.App/SSDRacer.App.csproj `
    -c Release -p:Platform=x64 -p:CpuArch=x64
```

`dotnet build` 编不了 `.vcxproj`，所以两步分开。

**坑**：MSBuild 编 C++ 那步的产物会掉进 `SSDRacer.TrackNative/SSDRacer.App/bin/...`（多一层目录），得手动拷到 `SSDRacer.App/bin/Release/net8.0-windows10.0.20348.0/x64/`。或者直接把 `TargetName` 的输出路径改对。

**坑**：改完代码 `dotnet build` 报 0 错误但产物没变，是增量判定没发现变化，加 `-t:Rebuild`。

**坑**：程序正在跑的时候编译会失败，报 `MSB3027` / 文件被锁定。提权启动的进程普通权限杀不掉，得用管理员身份杀。

---

## 已知限制

- 只认「性能」页的大图表。页面被别的窗口挡住就抓不到。
- 任务管理器最小化、或者图表的合成层没刷新时，拿到的是旧帧，车会在原地等着。
- 曲线陡升成一个几乎垂直的坡时，物理上是能冲上去的，但视觉上会像车贴着墙往上爬。这是高度场方案的固有毛病，不是 bug。
- 高 DPI 下必须 PerMonitorV2，否则坐标会错位。

---

## 来源与授权

这个项目派生自 **[CS-LX/CPURacer](https://github.com/CS-LX/CPURacer)**，原理、架构、导航与取帧思路都来自原作，原作者是 **CS-LX**。

原作仓库**没有附带任何 LICENSE 文件**，按著作权法默认「保留所有权利」。所以严格说，这份代码是未经明确授权的衍生作品。

在这里如实说明，是因为不想把来历藏起来。如果原作者对这份二次开发有意见，请联系删除。

改动部分：载具尺寸与外观、翻车判定（改成车顶碰曲线）、抓地力与车速、页面限制（原作只认 CPU 页，这里任意性能页都能跑）、全量改名与中英文案。