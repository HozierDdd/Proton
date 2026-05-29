**Per-game workaround config file**

在收集了Workaround和其形式之后, 需要进行的分析
1) workaround的pattern是什么 (开发者是怎么做的)
2) 为什么需要这个workaround (bug产生的原因)
3) workaround是否是通过bug report报告的
4) 如果有很多workaround是通过bug report报告的的话, 我们可以看看workaround解决的symptoms是什么?
5) 需要Pre-game workaround设置的组件的特性是什么?与之相反的组件的特性是什么?

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

