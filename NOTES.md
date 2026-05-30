**Per-game workaround config file**

在收集了Workaround和其形式之后, 需要进行的分析
1) workaround的pattern是什么 (开发者是怎么做的)
2) 为什么需要这个workaround (bug产生的原因)
3) workaround是否是通过bug report报告的
4) 如果有很多workaround是通过bug report报告的的话, 我们可以看看workaround解决的symptoms是什么?
5) 需要Pre-game workaround设置的组件的特性是什么?与之相反的组件的特性是什么?

Investigate Prompt:
帮我调研一下xx(submodule). 任务是: 
1) submodule的功能是什么;
2) 帮我调查一下有没有pre-game workaround;
3) 帮我调查一下有没有其他任何形式的workaround (非game- oriented);
4) 帮我调查一下有没有任何form的code fix的记录;
5) 这个组件在启动Windows游戏时, 所处于调用链的位置和证据.

注意: 不要更改我的文件, 请将调研结果放在对话框中即可. 

### DXVK

config file address:
dxvk/src/util/config/config.cpp

### dav1d

dav1d 是 AV1 视频解码库(由 VideoLAN/FFmpeg 团队开发),它的职责是解码 AV1 视频码流,和「游戏」「图形 API 翻译」完全没有关系。它在 Proton 里只是用来解码视频(比如游戏过场动画、媒体播放),因此根本不存在按游戏匹配的 app profile 这种机制。dav1d 只有运行时解码参数,通过 API(Dav1dSettings 结构体)或命令行工具传入,例如线程数、帧延迟、是否应用胶片颗粒等。这些是调用方在初始化解码器时设置的,不是按游戏/可执行文件自动匹配的配置文件。

### dxvk-nvapi

dxvk-nvapi 是 NVAPI 的开源重实现(让 Linux/Proton 下的游戏能调用 N 卡 NVAPI 功能,如 DLSS、Reflex 等)。它确实有针对特定游戏的 workaround,但不是 DXVK 那种「正则 → 配置项」的集中式 profile 列表,而是散落在代码里、直接用 exe 名硬编码的 if 判断。

Example:

```
bool needsPascalSpoofing(NV_GPU_ARCHITECTURE_ID architectureId) {
        if (architectureId >= NV_GPU_ARCHITECTURE_TU100 && getExecutableName() == std::string("MonsterHunterWorld.exe")) {
            log::info("Spoofing Pascal for Turing and later due to detecting MonsterHunterWorld.exe (Monster Hunter World)");
            return true;
        }
```
解释:

如果当前显卡架构是 Turing 或更新，并且正在运行的游戏是 MonsterHunterWorld.exe，就把显卡“伪装成 Pascal 架构”

什么是Nvidia显卡架构? 显卡型号, 如GTX 1080, RTX 2080, etc. 背后属于不同的 GPU 架构世代：

Pascal

GTX 1080、1070、1060

Turing

RTX 2080、2070、2060


**为什么DXVK中workaround的游戏数量和dxvk-nvapi中workaround游戏的数量差距这么大呢?**
DXVK 是所有 DirectX 游戏的必经渲染翻译层,要复刻几十年充满未定义行为的 D3D 语义,还要替游戏抹平 GPU 厂商差异,暴露面极大;而 dxvk-nvapi 只是少数现代游戏才用的、N 卡专属的信息/功能辅助 API,不碰渲染主流程,出错方式少、使用者也少。所以前者有 254 条、后者只有约 6 条,差距是由二者在系统中的角色本质决定的。

### FEX

FEX 是一个用户态的 x86 / x86-64 指令集模拟器(binary translator),让原本只能在 Intel/AMD 处理器上运行的 x86 程序,能在 ARM64(AArch64)等非 x86 架构的 Linux/Windows 上运行。

FEX 的 per-game workaround = 「按程序名把配置文件路由出来(可能多份),再连同其它配置来源一起按固定优先级合并成最终生效配置」。它是一套通用分层配置系统,per-game 只是优先级链中的几层,匹配方式是精确文件名而非正则。

相关文件位置: FEX/FEXCore/Source/Interface/Config/Config.cpp

然而, 这个 Config.cpp 本身不存放任何 workaround,它是 FEX 的「配置引擎/框架」。 workaround 的数据在 AppConfig/*.json 里;这个文件负责算出该读哪个 JSON、把多个来源按优先级合并、再把最终值提供给模拟器内核。

具体来说, GetApplicationConfig() 决定「去哪个 JSON 找这个游戏的 workaround」;
MetaLayer + LoadOrder 决定「游戏专属配置覆盖全局默认」;
Get/GetConv 把最终值交给模拟器内核执行。
真正的 workaround 数据在 Data/AppConfig/*.json,JSON 解析在 Source/Common/Config.cpp 的 AppLoader,这个文件只是把三者粘起来的引擎。

**既然FEX有程序名配置路由的话, 为什么DXVK还需要一个pre game workaround的配置文件呢?**

FEX 的程序名路由只能调FEX 自己(CPU 模拟层)的行为,而 DXVK 的 per-game workaround 调的是图形 API 翻译层的行为——FEX 既看不懂也碰不到这些旋钮;更重要的是,绝大多数玩家的栈里根本没有 FEX(只有 ARM 才有),所以 DXVK 必须自带一套可移植、自给自足的 per-game 配置,而不能寄生在 FEX 上。

### FFmpeg

FFmpeg 是通用媒体编解码库,它的输入是「媒体码流」,根本不知道也不关心是哪个游戏/进程在调用它。所以它的 workaround 不可能按游戏匹配。

但是 FFmpeg 有一套自己的 workaround 机制,只是匹配维度不同:它按「生成这段码流的编码器身份和版本」来匹配,而不是按消费者(游戏)。

### glslang

Glslang 是 OpenGL ES 和 OpenGL 着色语言的官方参考编译器前端。在本 Proton 项目语境里，它的作用就是把着色器源码编译/验证成 SPIR-V (Standard Portable Intermediate Representation - V, 是由 Khronos Group 维护的一种开放、跨 API 的二进制中间表示（IR）标准，专门用于表示图形着色器（Shaders）和通用并行计算内核（Compute Kernels）)，供 Vulkan 管线（DXVK/VKD3D 等）使用。

### graphene

GNOME 生态的「图形库专用数学类型」轻量层。只提供 2D/3D 变换/投影需要的数学基元,不碰窗口、绘制、场景图、输入。在 Proton 里它是 gstreamer / gst-plugins-base 的依赖(gstreamer/subprojects/graphene.wrap,被 GL 滤镜如 gstgltransformation 用到),属于媒体管线侧,和「游戏」「图形 API 翻译」无关。

**没有Pre-game workaround**

**其他形式(非 game-oriented)的 workaround**

没有运行时开关: 无 getenv、无 GRAPHENE_* 环境变量、无运行时 CPU 检测(无 cpuid / __builtin_cpu_supports)。唯一的自适应是 SIMD 加速路径,且是编译期决定的——graphene-simd4f.h 按 GRAPHENE_USE_SSE / SSE4_1 / AVX / ARM_NEON / SCALAR 走不同实现,这些宏在 meson 配置时按目标平台确定(可用 -Dsse2=false 等关闭)。本质是「编译期选平台最优实现,没有就回退 scalar」,是可移植性 fallback,不是 runtime workaround。

### gst-plugin-rs

gst-plugins-rs 是「GStreamer 的 Rust 插件集」,在 Proton 中实际只贡献 dav1d AV1 解码元件;它和 dav1d 本体一样,**没有 pre-game workaround**;唯一与游戏运行相关的「workaround」是 wine/winegstreamer 按元件名 "Dav1d" 强制设置 n-threads(在本组件之外);code fix 仅以上游 git 提交形式存在(refcount hack、缓冲池/分配查询修复等);其调用链位置是媒体解码支链,经由 Media Foundation/DirectShow → winegstreamer → GStreamer decodebin 自动插拔到 dav1ddec。

**启动 Windows 游戏时所处调用链位置**

Windows 游戏
  → 调 Media Foundation / DirectShow 播放媒体
  → wine winegstreamer 翻译为 GStreamer 管线
  → wg_media_type.c 把 MFVideoFormat_AV1 映射成 caps "video/x-av1"
  → GStreamer decodebin 按 caps 自动插拔(autoplug)解码器
  → 选中 libgstdav1d.so 的 dav1ddec 元件 (来自 gst-plugins-rs)
  → deep_element_added_cb → set_dav1d_n_threads() 配置线程数
  → 解码出的视频帧回传给游戏

### gstreamer

GStreamer 是一个基于「管线（pipeline）+ 元素（element）」的通用流媒体框架——你把 source → demuxer → decoder → converter → sink 这些元素串成一条管线，数据（音视频流）就在里面流过被处理。

在 Proton 语境里它的具体职责：给 Wine 的多媒体 API 提供后端解码能力。Windows 游戏调用 DirectShow / Media Foundation / WMV 播放过场动画、背景视频、音频时，Wine 自己不解码，而是把请求转交给 GStreamer 管线去 demux/decode（wg_parser.c 里用的是 decodebin 自动插件协商）。

它还是一堆媒体相关子模块的「宿主」——graphene、dav1d、FFmpeg、libsoup 等很多都是它 subprojects/*.wrap 里的依赖。

**媒体指什么?** 在游戏开发和游戏引擎（如 Unreal Engine、Unity）的语境下，“媒体（Media）”是一个特指的技术名词。它不是指游戏媒体（如 IGN、GameSpot 等新闻媒体），而是指“所有非实时生成的、需要从外部文件或流中加载并播放的动态音视频资产”。

**winegstreamer和gstreamer之间的关系是什么?**

一句话：gstreamer 是底层多媒体引擎（被调用方），winegstreamer 是 Wine 里调用这个引擎的「翻译适配层」（调用方）。前者是上游通用框架，后者是专门为了让 Windows 媒体 API 能跑在 Linux 上而写的胶水 DLL。

**Pre-game workaround —— 有**

但注意：不在 gstreamer 框架里，而在消费者 winegstreamer 里。机制类似 dxvk-nvapi（硬编码 if 判断），但匹配维度不是 exe 名，而是 getenv("SteamGameId") 比对 Steam AppID。例如:

(a) Biomutant (597820) — wine/dlls/winegstreamer/media_source.c

**原因**:（注释里写得很清楚）：UE4 的 FWmfMediaSession::GetEvents() 每帧检查 Time > CurrentDuration 来决定停止会话。Biomutant 预告片视频流(1:27.734)和音频流(1:27.776)时长不一致，而游戏只用视频流，导致时间永远追不上 media source 时长，开场动画后永久挂起。这个 hack 把时长缩短一点点满足条件。

### icu

这个repo感觉有问题, 似乎并没有这个submodule, 而且也没有pre-game workaround

### kaldi

Kaldi 是一个 C++ 语音识别(ASR, Automatic Speech Recognition)工具包.

**有没有 pre-game(per-game)workaround —— 有**

虽然 kaldi 引擎本身没有按游戏匹配的机制,但在它上游的 Wine 胶水层(windows.media.speech unixlib)里,存在硬编码按 SteamAppId 匹配的 per-game workaround,而且全部是为 Phasmophobia(SteamAppId = 739630) 准备的。这与 NOTES.md 里 dxvk-nvapi 的「散落在代码里、用 exe/appid 硬编码 if 判断」属于同一种 pattern(而非 DXVK 那种集中式正则 profile 表)。


