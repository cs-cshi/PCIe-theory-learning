本章描述物理层的链路训练和状态机（Link Training and Status State Machine, LTSSM）：
- 链路的初始化过程是链路从通电或复位到链路达到完全运行的 L0 状态，在此期间发生正常的数据流量。
- 链路电源管理状态 L0s、L1、L2 和 L3 以及状态转换。
- 阐述恢复状态，在此期间的位锁定、符号（Symbol）锁定或块锁定。
- 链路带宽管理的链路速度和位宽变化。

# 1. Overview
链路初始化和训练是由物理层控制的，基于硬件（非软件）的过程。该过程配置和初始化设备的链路和端口，使链路上能进行正常的数据报文通信。

<center>Figure 14-1: Link Training and Status State Machine Location</center>
![](./images/14-1.png)
如图 14-1 所示，完整的训练过程在复位后由硬件自动启动，并由 LTSSM（链路训练和状态机）管理。在链路初始化和训练过程中包括了几项步骤，先简单介绍下它们并引入一些术语：
- **Bit Lock**：链路训练开始时，接收端时钟尚未与输入信号的发送时钟同步，无法可靠地对输入位进行采样。在链路训练期间，接收端 CDR（Clock and Data Recovery）逻辑使用传入的比特流作为时钟参考信号来重新创建发送端的时钟。一旦从比特流中恢复了时钟，就可以说接收端完成了 bit lock，然后能够对输入的数据位进行采样。
- **Symbol Lock**：对于 8b/10b 编码，下一步是获取 Symbol Lock。同 bit lock 类似，虽然接收端现在可以识别各个位，但不知道 10 bit Symbol 的边界在哪里。当发送端与接收端交换 TS1 和 TS2 时，接收端在比特流中搜索可识别的模式来检测。最直接的是 Gen1/Gen2 中的 COM，其是特殊的编码方式，易于识别。识别到 COM 后，接收端不仅定位到两个 Symbol 的边界，也定位到两个有序集的边界。
- **Block Lock**：对于 128b/130b 编码，该过程与 Symbol Lock 略有不同，此时没有 COM 字符。接收端仍然可以通过其他容易识别的标识来找到边界。在 Gen3 中通过使用 EIEOS（Exectrical Idle Exit Ordered Set） 来定位，其使用 00h 和 FFh 字节交替模式。它定义了块边界，因为根据协议，当该有序集结束后，下一个 Block 必须紧随其后开始。
- **Link Width**：具有多 lane 的 PCIe 设备可以使用不同的链路宽度。例如，具有 x2 端口的设备可以与 x4 端口的设备相连。在链路训练期间，两个设备的物理层会测试链路，并将链路宽度设为彼此支持的最大值。
- **Lane Reversal**：多通道设备端口上的通道从 0 开始按递增顺序编号。通常一个设备端口的 lane0 连到对端设备 lane0，lane1 连 lane1，以此类推。然后有时希望能在逻辑上反转通道编号，以简化布线，使走线不必交叉，如图 14-2。只要一台设备支持通道编号反转功能，就可颠倒 lane 的连接关系。Spec 不强制要求支持此功能，因此电路板设计人员需要验证至少一个连接设备支持此功能，才能颠倒通道顺序布线。
<center>Figure 14-2: Lane Reversal Example (Support Optional)</center>
![](./images/14-2.png)
- **Polarity Inversion**：两个设备的 D+、D- 差分对也可以根据需要反转，以使电路板布局布线更容易。每个接收端 lane 必须单独检查差分信号连接情况，并在极性翻转的情况下，在训练期间根据需要自动纠正，如图 14-3 所示。接收端通过检测输入 TS1 和 TS2 的 Symbol 6~15 来实现这一点。如果接收端在 TS1 中收到了 D21.5 而不是 D10.2，或者在 TS2 中收到 D26.5 而不是预期的 D5.2，则代表当前通道差分对极性反转，必须纠正。与 lane 反转不同，极性反转的支持是强制的。
<center> Figure 14-3: Polarity Inversion Example (Support Required) </center>
![](./images/14-3.png)
- **Link Data Rate**：在复位后，链路初始化和训练状态机会先使用默认的 Gen1 速率，以实现向后兼容。如果链路宣称可以实现更高速率，那么训练过程中会通告双方自己支持的最高速率，并且在训练完成后，LSTTM 会自动重新进行一次过程更短的训练，以将链路速率改为双方都支持的最高速率。
- **Lane-to-Lane De-skew**：走线长度变化和其他因素会导致多 lane 链路的并行比特流在不同时间到达接收端，这一问题称为信号偏斜（signal skew）。接收端需要通过延迟较早到达的通道，来补偿通道间信号抵达时间差异，以对齐比特流（Gen1 允许到达时间有 20 ns 的差异）。这使得电路设计人员摆脱了有时难以创建等长走线的限制。

# 2. Ordered Sets in Link Training
## 2.1 General
不同类型的 Ordered Sets：
- TS1/TS2 Ordered Set
- Electrical Idle Ordered Set ( EIOS)
- FTS Ordered Set (FTSOS)
- SKP Ordered Set (SOS)
- Electrical Idle Exit Ordered Set (EIEOS)
TS1 和 TS2 在训练过程中很重要，Gen1/Gen2 下其格式如图 14-4 所示
<center>Figure 14-4: TS1 and TS2 Ordered Sets When In Gen1 or Gen2 Mode</center>
![Figure 14-4: TS1 and TS2 Ordered Sets When In Gen1 or Gen2 Mode](./images/14-4.png)

Gen3 模式如图 14-5 所示
<center>Figure 14-5: TS1 and TS2 Ordered Sets When In Gen3 Mode</center>
![Figure 14-5: TS1 and TS2 Ordered Sets When In Gen3 Mode](./images/14-5.png)

## 2.2 TS1 and TS2 Ordered Sets
正如上节如中描述的，TS1/2 由 16 个 Symbol 组成。在 LTSSM(Link Training and Status State Machine) 的 Polling、Configuration and Recovery states 状态期间，通信双方会交换 TS1/TS2。为了便于描述，PAD character 用于描述填充字符，Gen1/Gen2 表示 K23.7，Gen3 由 data byte F7h 表示。

table 14-1 总结了 TS1 的内容摘要，其详细描述如下：
- Symbol 0:
	- Gen1/Gen2，任何 Ordered Set 第一个 Symbol 是 K28.5(COM)。能用来实现 Symbol Lock 和 lane 间 de-skewing。
	- Gen3，块前两位 2-bit 是 Sync Header，表中未给出该两位。其后第一个 Symbol 标明是哪种 Ordered Set，TS1 是 1Eh，TS2 是 2Dh。
- Symbol 1 (Link #)：链路编号，在 Polling 状态下，该字段使用填充字符 PAD 填充，其他状态为被分配的编号（Link Number）
- Symbol 2 (Lane #)：通道编号，在 Polling 状态下，该字段使用填充字符 PAD 填充，其他状态下为被分配的编号（Lane Number）
- Symbol 3 (N_FTS)：表示当前速率下，接收方从 L0s 电源状态退出返回 L0 状态所需要接收的的快速训练序列（Fast Training Sequence, FTS）数量。在退出 L0s 状态时，发送端至少会发送 N_FTS 个 FTS。此过程所需的时间取决于所需的 FTS 数量以及当前的链路速率。举例来说，Gen1 速率下 4ns/Symbol，如果需要 200 FTS，那么就是 $200 FTS * 4 Symbols per FTS * 4ns/Symbol = 3200ns$。如果在发送端置位了扩展同步位（Extended Synch bit），则需要发送 4096 个 FTS，旨在为外部链路观测工具提供足够的时间以获取位和 Symbol 锁定。
- Symbol 4 (Rate ID)：设备报告其所支持的数据速率，以及一些提供给由硬件发起的带宽更改功能的信息。所有设备必须支持 Gen1，且在复位后链路始终自动从 Gen1 速率开始，以便向后兼容旧设备。如果设备支持 Gen3，那么也需要支持 Gen2。除此之外，该符号中还包括的其他信息有：
	- Autonomous Change：当该比特置为 1 时，表示任何请求带宽更改的请求都是基于电源管理方面的原因而发起的。如果在发出带宽改变请求时，该比特未置为 1，则表示在更高速率或更宽链路上检测到工作不稳定的情况，需要改变带宽配置（降低速率或者减小链路宽度）来解决这些问题。
	- Selectable De-emphasis：
		- 上游端口（Upstream Ports）设置该域，表明 Gen2 de-emphasis 级别，具体如何选择取决于具体实现。在 `Recovery.RcvrCfg` 状态下，设备锁存接收到该域的值，存放在设备内部。（spec 中指出该值存储在 `select_deemphasis` 变量中）
		- 下游端口（Downstream Ports）和根节点端口（Root Ports）：在 `Polling.Compliance` 状态中，基于接收到的该比特值，设置 `select_deemphasis` 变量。在 `Recovery.RcvrCfg` 状态中，发送端在其 TS2 中设置该位，其值基于 `Link Control 2` 寄存器中的 `Selectable De-emphasis` 字段。由于该寄存器位是硬件初始化的，因此可以在上电阶段后通过固件配置恰当的值，或者将其硬件初始化值设置为一个保守但安全（Strapping）的数值。
		- 在 Gen2 的 Loopback 模式下，Slave（从）机的 de-emphasis 值由 Master 机发送的 TS1 中的该字段对应比特设置。
	- Link Upconfigure Capability：报告降位宽的链路在减少链路的宽度后，能否恢复到原来链路宽度。如果在 `Configuration.Complete` 状态中，双方设备都报告了自己有这个能力，那么会当作共识记录下来。如设置 `upconfigure_capable` 位。
<center>Table 14-1: Summary of TS1 Ordered Set Contents</center>
![](./images/table14-1-1.png)
![](./images/table14-1-2.png)
![](./images/table14-1-3.png)
- Symbol 5 (Training Control)：交流链路双方的一些特殊情况，比如热复位、启用 Loopback、关闭链路、关闭扰码等。
- Symbols 6-9 (Equalization Control)：
	- 对于 Gen1/Gen2，Symbol 7-9 仅是 TS1/TS2 的标识符，Symbol 6 通常也只是标识符，此时 Symbol 6 的 bit 7 为 0。但如果 该比特为 1，则代表这是从下游端口（DSP，面向下游的端口，如 Root Port）发送的 EQ TS1/TS2。其中EQ 标签表示均衡（Equalization），发送 EQ TS1/TS2 意味着链路速率将提升为 Gen3，此时上游端口（USP，面向上游的端口，如 Endpoint Port）需要知道当前使用的均衡参数。对于 EQ TS1/TS2，Symbol 6 提供给 USP 的信息包括发送端 Presets 和接收端预设集选择提示（ Presets  Hint）。支持 Gen3 的端口必须能接收任何 TS 类型（reqular or EQ），但不支持 Gen3 的端口不需要接收 EQ 类型。这些 Presets 的可能值会在后续列出。
	- 对于 Gen3，Symbol 6-9 提供均衡过程的 Preset 和 Coefficients。TS2 的 Symbol 6 的 bit 7（Request Equalization 域）现在被用于申请重新进行均衡。此时，Symbol 6 的 bit 6（Quiesce Guarantee 域）可能也需要被置起，表示只要重新均衡的速度够快，那么所耗费的时间不会导致诸如完成超时等问题（返回 L0s 1 ms 内）。在有些场景会使用到这个功能，如在使用均衡得到的设置时检测到了问题。DSP 可以使用 bit 6/7 要求 USP 来发起重新均衡的请求，并保证重新均衡不会带来副作用，尽管 USP 无需对此作出响应。后续会详细介绍链路均衡过程。
- Symbols 10-13：TS1 or TS2 标识符。
- Symbols 14-15 (DC Balance，发送 0/1 数量平衡)：
	- 对于 Gen1/2，这只是 TS1/TS2 标识符，因为 DC 均衡由 8b/10b 编码完成。
	- 对于 Gen3，这两个 Symbol 的数值取决于 Lane 的 DC 均衡需要。发送端的每个 Lane 必须独立追踪当前所有正在被发送的加扰后的 TS1 和 TS2 的实时 DC 均衡（Running DC Balance）情况。实时 DC 均衡（Running DC Balance）指发送的 1 和 0 之间的数量差，各个 lane 必须能够收发方向追踪 0 和 1 的数量差值，最大支持差值为 511。这些计数器在记满后保持不变，但是会更新差值减小的变化。如计数器记录到 1 比 0 多 511 个，此后无论发送多少个 1，计数器仍停留在 511，然后当有 0 时计数器会减少。当发送 TS1/TS2 时，Symbol 14、15 的数值由以下算法决定：
		- 如果在 Symbol 11 结束时，Running DC Balance value > 31，且 1 数量更多，那么 Symbol 14 = 20h, Symbol 15 = 08h。如果 0 更多，Symbol 14 = DFh, Symbol 15 = F7h。
		- 如果 Running DC Balance value > 15，Symbol 14 = the normal scrambled TS1 or TS2 identifier，Symbol 15 = 08h 以减少 DC Balance 中 1 的数量，或者 Symbol 15 = F7h 以减少 0 的数量。
		- 否则，将发送 normal TS1 or TS2 identifier 标识符符号。
	- 关于 Gen3 DC Balance 需要注意的事项：
		- DC 均衡计数器（The Running DC Balance）将在退出电气空闲状态时，或在一个数据块之后收到 EIEOS 时复位；
		- The DC Balance Symbols 将绕过加扰，以确保发送的是预期的符号。

<center>Table 14-2: Summary of TS2 Ordered Set Contents</center>
![](./images/table14-2-1.png)
![](./images/table14-2-2.png)

# 3. Link Training and Status State Machine(LTSSM)
## 3.1 General
<center>Figure 14-6: Link Training and Status State Machine (LTSSM)</center>
![](./images/14-6.png)
上图描述了 LTSSM 的高层次抽象结构。每个状态又划分为若干子状态。在基础复位（Fundamental Reset），即冷复位 (Cold Reset）和暖复位（Warm Reset)，或者热复位（Hot Reset）释放后，进入的第一个状态是 `Detect` 状态。

Cold Reset：
- 定义：冷复位是一种完全的重置，它将设备恢复到初始状态。这包括清除所有寄存器、状态和配置信息，类似于设备刚刚上电时的状态。
- 触发方式： 冷复位通常由硬件或系统引导过程中的固定电源控制（如物理电源开关）触发。
- 影响： 冷复位将导致设备重新进行完整的初始化，可能需要较长的时间来完成。
Warm Reset：
- 定义： 温复位是一种部分的重置，它不会清除所有设备的配置信息，而是保留一些初始化状态。
- 触发方式： 温复位通常由设备内部或系统软件发出的命令触发，而不涉及硬件电源的改变。
- 影响： 温复位相对较快，因为它不需要重新初始化所有的配置信息。
热复位（Hot Reset）：
- 定义： 热复位是一种特殊的复位类型，它允许PCIe设备在不中断总线上其他设备的通信的情况下进行重置。
- 触发方式： 热复位是通过发出特殊的控制命令来触发的，而不会中断总线上的其他设备。
- 影响： 热复位的目标是在不影响其他设备的通信的情况下重新初始化设备。

LTSSM 包含 11 个顶级状态，分别事：Detect, Polling, Configuration, Recovery, L0, L0s, L1, L2, Hot Reset, Loopback, and Disable。他们可以分为 5 类：
- 链路训练状态，Link Training states
- 重训练状态，Re-Training (Recovery) state
- 软件驱动的电源管理状态，Software driven Power Management states
- 主动电源管理状态，即硬件驱动的电源管理状态，Active-State Power Management (ASPM) states
- 其他状态
在任意复位释放后，LTSSM 即进入了链路训练状态（Link Training states），一切正常的话，会按照 Detect => Polling => Configuration => L0 的顺序跳转状态。待进入 L0 状态后，即可以进行正常的数据报文收发操作。

链路重新训练也称为恢复状态，进入该状态的原因有很多，如从低功耗链路状态恢复，或切换链路带宽（speed or width changes）。在恢复状态下，链路会重复类似于训练状态的操作，来解决链路中的问题，并最终回到 L0，这一正常工作状态。

设备中功耗（电源）管理软件可以将设备置于低功耗设备状态（D1, D2, $D3_{Hot}$ , $D3_{Cold}$ ），这样的操作会强制链路进入较低的电源管理链路状态（L1 or L2, Power Management Link state）。注意区分低功耗设备状态（low-power device state）和低功耗链路状态（low-power Link state）。

如果链路上长时间都没有报文需要发送，ASPM 硬件逻辑会使链路自动进入低功耗 ASPM 状态（L0s or ASPM L1）。

此外，软件可以直接使链路进入一些其他状态，Disabled, Loopback, or Hot Reset。这里统一称为其他状态。

## 3.2 Overview of LTSSM States
下面将简要描述 11 个顶级状态。
- **检测状态 Detect：** 复位释放后的初始状态。在此状态下，本方设备从电气特性的角度，检测链路对端设备是否存在。串行传输通常不用检测对端是否存在，但 Detect 状态便于测试。除了复位释放之外，还可以从许多其他 LTSSM 状态进入 Detect。
- **轮询状态 Polling：** 该状态下发送端以 Gen1 速率向对端发送 TS1/TS2，使用协议最低速率以实现对早期协议的向后兼容。便于接收端可以使用接收的 TS1/TS2 序列完成下述功能：
	- Achieve Bit Lock，位锁定
	- Acquire Symbol Lock or Block Lock，符号（Gen1/2）、块（Gen3）锁定
	- Correct Lane polarity inversion, if needed，校正通道极性反转
	- Learn available Lane rates，得知通道支持的速率
	- 根据需要进行合规性测试序列（Compliance test sequence）：当检测状态下检测到接收端，但没有返回任何流量，则可以理解为对端设备是一个测试负载（Test load）。此时，发送方会发送指定的合规性测试图样（Pattern）以方便测试。这项特性能够使测试设备快速验证电压、BER（Bit Error Ratio）、时序和其他参数是否在链路容差范围之内。
- **配置状态 Configuration：** 上游（Upstream）和下游（Downstream）组件以 Gen1 速率交换 TS1/TS2，以实现以下目标：
	- Determine Link width，协商决定链路宽度
	- Assign Lane numbers，为各通道指派编号
	- Optionally check for Lane reversal and correct it，检测通道是否需要顺序或者极性交换
	- Deskew Lane-to-Lane timing differences，补偿各个通道之间的偏斜
	- 该状态开始可以关闭加扰，此时可以进入 Disable 或者 Loopback 状态，并从 TS1、TS2 序列交换时达成共识的 N_FTS，即从 L0 状态转变到 L0s 状态所需的 FTS 有序集数量。
- **L0：** 链路全功能正常运行的状态，此时链路上会进行正常的 TLP、DLLP 报文和有序集交换。此状态下链路速率可以高于 Gen1，但只能在进入 Recovery 状态，经历一次链路速率变化程序之后，才能切换到更高的速率。
- **恢复状态 Recovery：** 当链路需要重新训练时进入此状态。这可能是以下原因导致：L0 状态中发生了错误、从 L1 恢复到 L0 、从 L0s 状态恢复到 L0 时，无法通过 FTS 序列重新完成训练。在恢复状态中，会重新进行 Bit Lock and Symbol/Block Lock，锁定的过程和 Polling 状态中时相同，但通常此次花费时间更少。
- **L0s：** L0s 是一个由硬件控制的 ASPM 低功耗，其目的是在节省一定功耗的同时，能够快速地恢复到 L0 状态。当链路一端发送在 L0 状态下发送 EIOS 序列时，会进入 L0s 状态。退出 L0s 状态时，会通过 FTS 序列来快速重新完成 Bit and Symbol/Block Lock。
- **L1：** 此状态比 L0s 有更大的节能效果，但恢复到 L0 时间比 L0s 更长。链路双方需要共同协商以进入到 L1，可以通过以下两种方式之一进入：
	- ASPM 控制下自动进入：当上游端口设备没有等待调度发送的 TLP/DLLP 报文时，硬件逻辑将自动与下游端口协商，一起将链路转为 L1 状态。如果下游端口同意，则链路进入 L1，否则上游端口将进入 L0s（if enabled）。
	- 另一种情况是功耗管理软件命令设备进入低功耗状态（$D1, D2, or D3_{Hot}$）。此时上游端口通知下游端口必须一起进入 L1 状态，下游端口会响应该通知，并进入 L1 状态。
- **L2：** 在此电源下，设备主电源被关闭以实现节约更大功耗。此时几乎所有逻辑都已关闭， $V_{aux}$ 电源仍可使用以供设备唤醒。拥有唤醒能力的上游端口可以发送低频 Beacon 信号，下游端口可以将其转发到 Root Complex，告知上层系统。通过 Beacon 信号或边带 WAKE# 信号，设备能够触发系统唤醒事件，使主电源恢复。（另外还有 L3 链路电源状态，它与 LTSSM 状态无关，L3 完全关闭且 $V_{aux}$ 不可用，无法唤醒）。
- **回环状态 Loopback：** 回环状态定义为一种测试状态，协议中没有详细定义接收方的行为（如接收方哪些逻辑会参与回环测试）。回环状态中基本行未很简单：回环发起方（Loopback Master）发送 TS1 有序集给回环接收方（Slave），并置位 TS1 训练控制（Training Control）字段中 Loopback bit。当接收方连续接收到 2 个 Loopback bit 置起的 TS1 序列后，进入回环状态，将接收到的任何内容都重新发送给发起方。发起方会检查接收到的内容，和先前发送的内容进行比较，如果内容经双方 8b/10b 编码（解码）后校验一致，说明链路可以通过环路验证，完整性没有问题。
- **禁用状态 Disable：** 此状态允许将链路配置为禁用状态，该状态下发送端处于电气空闲状态，接收端处于低阻抗状态。在某些情况下，比如链路变得不可靠或者对端设备被意外移除，此时 Disable 是表示这些意外情况的必要状态。另外，软件也可以通过链路控制寄存器 （Link Control register） 中的禁用比特（Disable bit），将设备配为禁用状态。设备被禁用后，会连续发送 16 个链路禁用比特（Disable Link bit）置位的 TS1 序列，其位于 TS1 的训练控制域（Training Control Field），通知接收方进入禁用状态。
- **热复位 Hot Reset：** 软件可以通过置位 Bridge Control 寄存器中的 Secondary Bus Reset bit 来复位链路。软件配置后，桥的下游端口（Downstream Port）会发送训练控制域（Training Control Field）热复位（Hot Reset）比特置位的 TS1 序列。（详情可见原文 P837 页的 "Hot Reset (In-band Reset)" 节）。当接收端收到两个连续置位了 Hot Reset bit 的 TS1 时，它必须复位自身设备。

## 3.3 Introduction, Examples and State/Substates
本章的剩余内容会对每个 LTSSM 状态进行介绍和讨论。基于每种状态不同的复杂度，讨论会包括一些介绍，通用的背景知识，部分状态/次状态还会有相应的示例。基于具体的需求，读者可以只浏览某个状态简介的部分，而跳过详细讨论的部分，本章组织结构完全支持读者这么做。

每个设备必须以 Gen1 速率执行初始链路训练。图 14-7 突出显示了初始训练中涉及的状态。能够支持更高速率的设备必须转换到 Recovery 状态，才能进行速率切换。
<center> Figure 14-7: States Involved in Initial Link Training at 2.5 Gb/s</center>
![](./images/14-7.png)

# 4. Detect State
## 4.1 Introduction
图 14-8 展示了与 Detect 状态相关的两个子状态和状态跳变。Detect 状态下的链路行为是发送方不断检测链路对端是否有接收方存在。因为 Detect 只有 2 个次状态，行为比较简单，本节直接讨论两个次状态的细节，不再讨论 Detect 状态本身了。
<center>Figure 14-8: Detect State Machine</center>
![](./images/14-8.png)

## 4.2 Detailed Detect Substate
### 4.2.1 Detect.Quiet
该次状态是所有复位（除功能级复位外，Function Level Reset, FLR）或上电事件（Power-up）后的初始状态。并且协议规定必须在复位之后 20ms 内进入该状态。当其他状态无法正常进入下一个状态时，也会进入该次状态，这些状态包括 Disabled、Loopback、L2、Polling、Configuration、Recovery。Detect.Quiet 次状态属性如下所示：
- 发送端以电气空闲的状态启动（此时 DC commom mode voltage 不需要在通常规定范围内）
- 目标数据速率为 Gen1 的 2.5GT/s。如果进入此状态时不是 Gen1 速率，则 LTSSM 必须在此子状态内等待 1ms，然后再将速率切换为 Gen1。
- 物理层的状态位为 0 时（LinkUp = 0），向数据链路层表示链路尚未就绪。LinkUp 状态位是一个内部信号（在标准配置空间中找不到），其还可以标识物理层是否完成链路训练（LinkUp = 1），从而通知数据链路层和流控逻辑（Flow Control）开始初始化。（参见 P223 页 “FC Initialization Sequence” 一节）。
- 任何先前的均衡（Eq., Euqalization）状态会在置位 4 个 2-bit 链路状态寄存器后清除，他们分别是：Eq. Phase 1/2/3 Successful 以及 Eq. Complete。
- Variables：
	- 几个变量会在 Detect.Quiet 状态清楚：
		- directed_speed_change = 0b
		- upconfigure_capable=0b
		- equalization_done_8GT_data_rate=0b
		- idle_to_rlock_transitioned=00h
		- select_deemphasis 变量的处理方式取决于端口类型
			- 对于 Upstream 端口，变量的值由硬件指定
			- 对于 Downstream 端口，变量的值使用链路控制（Link Control 2）寄存器中保存的 selectable Preset/De-emphasis  的值
	- 由于上述变量从PCIe 2.0 协议之后开始引入，更早期协议的设备不存在这些值，对于这些设备来说，他们的行为相当于：
		- directed_speed_change=0b
		- upconfigure_capable=0b
		- idle_to_rlock_transitioned=FFh

Exit to "Detect.Active"：LTSSM 会在 12 ms 超时或者任意一个通道退出电气空闲有状态后，转入下一个次状态。

### 4.2.2 Detect.Active
该子状态从 `Detect.Quiet` 进入。在 Detect.Active 状态中，发送端通过将直流共模电压（DC common mode voltage）设置为一个合法范围内的任意电压值，然后改变它来测试每个 lane 上是否有接收端连接。其检测逻辑是观察每个 lane 电压充电所需时间的变化（电压变化速率），并将以与对照值比较。这个对照值一般是没有接收端连接的情况，如果有接收端连接，那么充电的速率会大大的降低，两者之间巨大的差异使检测结果十分容易观察到（详见 P460 "Receiver Detection"）。为了便于表示，下文中将本状态中检测到的接收端 lane 称为 检测完成的通道“Detected Lanes”。
- Exit to "Detect.Quiet"
	- 如果没有任何一个 lane 检测到接收端时，将返回到 `Detect.Quiet`。只要没有检测到接收端，状态机会一直每隔 12ms 在 Quiet、Active 两个次状态之间进行一个循环。
- Exit to "Polling.State"
	- 如果在所有 lane 上都正常检测到接收端，则进入下一状态 Polling。此时 lane 需要驱动协议规定的 0-3.6V（ $V_{TX-CM-DC}$）范围内直流共模电压（DC common voltage）。
- Special Case:
	- 如果设备只有部分 lane 检测到接收端（如 X4 设备连接到 X2 设备），那么会等待 12 ms 再进行一次检测。如果检测的结果仍然相同，那么进入到 Polling 状态，否则返回 `Detect.Quiet`。在 Polling 状态中，如果存在没有接收端的 lane，有两种可能的处理方式：
		1. 如果 lane 可以作为单独的 Link 运行（see "Designing Devices with Links that can be Merged"），则使用另一个 LTSSM 对这些 lane 重复检测过程。
		2. 如果不存在另一个 LTSSM，那么这些 lane 不会成为链路的一部分，必须设置为电气空闲状态。

# 5. Polling State
## 5.1 Introductin
LTSSM 进入轮询状态时，链路处于电气空闲状态，不过，轮询状态期间会在链路两端之间交换 TS1 和 TS2 有序集。**轮询状态的主要目的是使链路两端设备能够听懂对方在说什么**，也就是说两端设备需要在彼此传输的比特流上，建立 bit 和 Symbol 的锁定，并解决极性翻转（polarity inversion）恢复之类的事宜。在这些工作完成后，设备双方都能正确接收对方的 TS1/TS2 ordered sets。图 14-9 显示了 Polling 状态机的子状态。
<center>Figure 14-9: Polling State Machine</center>
![](./images/14-9.png)
## 5.2 Detailed Polling Substates
### 5.2.1 Polling.Active
**During Polling.Active**
一旦链路双方的共模电压稳定在发送余量域（Transmit Margin field）允许的范围内之后，发送端会发送至少 1024 个连续的 TS1 有序集。因为链路双方退出 `Detect` 状态的时刻可能不同，因此 TS1 命令序列的交换是不同步的。在 Gen1 速率下，发送 1024 个 TS1  花费时间是 $64\mu s$。

Active 状态下，有一些值得注意的事项：
- 必须在 TS1 序列的通道（Lane）和链路编号（Link number）字段中使用填充字符 PAD。
- 设备需要通告（advertise）自己所支持的所有速率，即使设备不打算使用这些速率。
- 接收端使用接收到的 TS1 序列实现 Bit Lock，然后 Gen1/2 实现 Symbol Lock，Gen3 实现 Block Alignment。

**Exit to "Polling.Configuration"**
在发送方发送完 1024 个 TS1 里最后一个 TS1 序列之后，如果所有检测到的 lanes 都收到了 8 个连续的训练序列（如果存在极性翻转的话，那么收到的就是训练序列的补集），并且这些训练序列满足以下条件之一的话，状态机进入 `Polling.Configuration` 状态：
- 接收到的 TS1 的 Link 和 Lane 字段都被设置为全填充字符（PAD），并且 `Compliance Receive` bit 设为 0b（bit 4 of Symbol 5）
- 接收到的 TS1  的 Link 和 Lane 字段都设置为 PAD，并且 `Loopback` bit 设为 1b（bit 2 of Symbol 5）
- 接收到的 TS2 的 Link 和 Lane 字段都设为 PAD
如果上述条件都不满足，那么在 24ms 的超时之后，在接收到某个 TS1 后，已经发送了至少 1024 个 TS1，只需要任意检测到的通道收到了 8 个 连续的TS1 或 TS2（或补集），且它们的 Lane、Link 编号是 PAD，并且满足以下条件之一（其实以下条件和超时前判断条件一致），那么状态机进入 `Polling.Configuration` 状态：
- 接收到的 TS1 的 Link 和 Lane 字段都被设为 PAD，并且 `Compliance Receive` bit 设为 0b（bit 4 of Symbol 5）
- 接收到的 TS1 的 Link 和 Lane 字段都被设为 PAD，并且 `Loopback` bit 设为 1b（bit 2 of Symbol 5）
- 接收到的 TS2 的 Link 和 Lane 字段都设为 PAD
如果在任何 lane 上都没有满足上述条件的情况出现，那么如果自从进入 `Polling.Active` 状态以来，有至少预定数量被检测到的 lane 上，还检测至少一次退出 `Electrical Idle` 的现象（这是为了防止一个或多个失效的发送端或接收端导致链路不能进行配置），那么也能进入到 `Polling.Configuration` 状态。此处提到的预定数量由具体实现决定，spec 1.1 中是必须检测到所有 lane 都退出 `Electrical Idle`。

**Exit to "Polling.Compliance"** 
如果 `Link Control 2` 寄存器中 `Enter Compliance` bit 被设为 1b，那么进入 `Polling.Compliance`。或者如果在进入 `Polling.Active` 状态之前，该比特已经被置位，那么将直接从 `Polling.Active` 状态进入到 `Polling.Compliance` 状态，不会在 `Polling.Active` 状态中发送任何 TS1 序列。

否则，等待 24ms 后，再次判断进入 `Polling.Compliance` 状态的条件：
- All Lanes 在进入 `Polling.Active` 状态后，都没有检测到对端退出 `Electrical Idle` 状态（意味着对端是一个被动测试负载，比如至少在一个 lane 上挂在了一个测试电阻，将强制使所有 lane 进入 `Polling.Compliance` 状态）
- 接收到 8 个连续的 TS1，其 Link、Lane 全被设为 PAD，并且 `Compliance Receive` bit 设为 1b（bit 4 of Symbol 5），且 `LoopBack` bit 设为 0b（2 bit of Symbol 5）

**Exit to "Detect State"** 
如果在 24ms 之后，没有满足进入 `Polling.Configuration` 状态，或者 `Polling.Compliance` 状态的条件，那么返回 `Detect` 状态。

### 5.2.2 Polling.Configuration
在 `Polling.Configuration` 状态中，发送方停止发送 TS1 序列，转而发送 TS2 序列，TS2 序列中的链路和通道 lane 字段仍然使用填充字段 PAD 填充。该状态中，发送方转而发送 TS2 的目的是通知链路对端设备，本方已经做好准备进入状态中的下一个状态。这是一种使链路双方设备同步 LTSSM 而设计的握手机制。双方设备都无法独自进入下一状态，除非链路双方设备都准备就绪。开始发送 TS2 序列是通知对端本方准备就绪的方式，所以一旦设备同时发送并且接收到 TS2 序列，代表双方和对端设备都已经就绪，设备可以进入下一状态。

**During Polling.Configuration** 
发送方在所有识别的 lane 上发送 TS2 序列，其链路和通道字段使用填充字段 PAD。并且发送方必须通知对端自己所支持的所有数据速率（Polling.Active 也要通知），即使是那些不打算使用的数据速率。此外，如果有需要的话，每个通道的接收方必须独立地恢复差分输入信号的极性。关于极性恢复的具体内容，参见原文 P506 页 “Overview” 小节。此外，Transmit Margin 字段必须重置为 000b。

**Exit to "Configuration State"** 
在任意通道上接收到 8 个连续的链路和通道（Lane）PAD 字段填充的 TS2 序列后，并且从接收到第一个 TS2 序列开始，已经发送了至少 16 个 TS2 序列后，退出 `Polling.Configuration` 状态进入 `Configuration` 状态。
	
**Exit to "Detect State"** 
如果上述条件在 48ms 的超时后仍未满足，将退出 `Polling.Configuration` 状态进入 `Detect` 状态。

**Exit to Polling.Speed(Non-existent substate)** 
此处讨论的情况是一项 1.0 协议历史中的预留设计，Polling 状态下属的次状态在协议 1.0 版本发布之后已经发生了改变。在当时，设计者觉得在未来新的，更快速率出现之后，有必要在 Polling 状态中尽可能快速地切换到最高速率。然后，更高的速率出现之后，设计者意识到相比在第一次静态配置状态中把速率配置到最高速率，不如动态地根据电源管理需求，动态地提高或者降低速率。由于在 Polling 阶段需要清除大量链路的相关配置，所以重新进入 Polling 状态进行动态速率调节是不适合的，因此动态速率调节的功能被移出了 Polling 阶段，而设置于 Recover 阶段中。具体可见图 14-10。
<center>Figure 14-10: Polling State Machine with Legacy Speed Change</center>
![](./images/14-10.png)
如今，LTSSM 在复位之后总是会将速率训练为 2.5 GT/s，哪怕支持更高的速率。在 LTSSM 状态机进入 L0 状态之后，一旦有更高的可用速率，状态机进入 Recovery 状态，并且尝试将速率训练为双方都支持的最高速率。双方可支持的最高速率会在双方交换的 TS1 和 TS2 序列中体现，所以任意一方都可以通过使用自身的状态机从 Recovery 状态，发起一次速率改变行为。协议中仍然会罗列 Polling.Speed 次状态，但会将其标注为不会进入（unreachable）的状态。

### 5.2.3 Polling.Compliance
`Polling.Compliance` 子状态用于测试。发送方会发送特定的码字组合（pattern），构建一种码间干扰和串扰接近最严重的场景，用于链路信号质量分析。在该子状态中，可以发送两类码字，分别是 Compliance Pattern 和 Modified Compliance Pattern。

**Compliance Pattern for 8b/10b** 
该码型由 4 个按顺序重复的符号组成：K28.5-、D21.5+、K28.5+ 和 D10.2-。其中 +/- 表示当前运行差异（current running disparity, CRD）。如果链路有多个通道，会在每个通道注入 4 个延时符号，其中 2 个注入在合规模式之前，2 个注入在之后，表 14-3 中用 D 表示。注意该符号不是同时注入，而是在上一个通道注入完成后，再在下一个通道注入，当延迟符号传播最后一个通道且注入完成后，从第 1 个通道起始位置继续。该移动传播符号能确保相邻通道的干扰，提供更好的测试条件。
<center>Table 14-3: Symbol Sequence 8b/10b Compliance Pattern</center>
![](./images/table14-3-1.png)
![](./images/table14-3-2.png)

**Compliance Pattern for 128b/130b** 
该码型由下述 36 块重复序列组成：
- 第一个块：同步头 01b，后接未加扰的 64 个 1，再接 64 个 0
- 第二个块：同步头 01b，后接 table 14-4 中未加扰内容，其中 P 表示使用的 Tx preset，~P 表示按位取反。
- 第三个块：同步头 01b，后接 table 14-5 中未加扰内容。
- 第四个块：EIEOS Block
- 后续 32 个块均是 Data Block，每一个块包含 16 个加扰的 IDL Symbol（00h）。
<center>Table 14-4: Second Block of 128b/130b Compliance Pattern</center>
![](./images/table14-4-1.png)
![](./images/table14-4-2.png)
<center>Table 14-5: Third Block of 128b/130b Compliance Pattern</center>
![](./images/table14-5-1.png)
![](./images/table14-5-2.png)
**Modified Compliance Pattern for 8b/10b** 
第二种合规模式增加了一个错误状态字段，用来记录在 `Polling.Compliance` 中接收端检测到的错误数量。

相比 `Compliance Pattern`，该种模式下码型由 8 个字符组成：K28.5-、D21.5+、K28.5+、D10.2-、ERR、ERR、K28.5-、K28.5+。2 个 error status 用来报告错误状态，并在最后增加 2 个 K28.5。
<center>Table 14-6: Symbol Sequence of 8b/10b Modified Compliance Pattern</center>
![](./images/table14-6-1.png)
![](./images/table14-6-2.png)
待补充...

# 6. Configuration State
设备初始化时，Configuration 状态在 2.5GT/s 速率下配置链路以及通道编号。Gen2/3 速率时，设备也可能从 Recovery 状态进入 Configuration 状态。此时状态转换的主要目的是为了进行多通道设备的链路位宽动态转换。动态转换仅支持 Gen1/2 速率的设备。详情见 P552 的 "Detailed Configuration Substates" 一节。

## 6.1 Configuration State - General
本状态主要目标是弄清楚设备端口（Port）和各个通道（Lane）的连接情况，以及为连接的 Lane 分配 Lane 编号。举例来说，对一个连接了 8 个 Lane 的端口来说，可能只有其中的 2 个 Lane 是可用的，再比如这些通道可以组合为多条链路，比如组合成两个 x4 的链路（每个链路有 4 个通道），Configuration 状态的目的即使为了搞清楚这方面的信息。另外，和其他状态不同的是，端口在此状态的行为区分为面向上游端口和面向下游端口两种情况，两类端口在 Configuration 状态中被定义为不同的角色。因此，在本状态中会分别讨论面向上游和下游通道的行为。

DSP（Downstream Port，向下游发送数据）端口在剩余的链路初始化过程中扮演”领导者“，而 USP（Upstream Port，向上游发送数据）端口的角色是”跟随者“。DSP 会为 USP 确定链路以及通道的编号，USP 则只是简单地重复他接收到的值给 DSP，除非出现了编号冲突的情况。我们会在本节后续内容中讨论这种意外情况。Configuration 状态中，双方设备交换的 TS1 序列中反映分配的链路以及通道编号，如图 14-13 红圈部分所示，这两个数值域在被分配具体的数值前，使用填充符号（PAD Symbol）作为占位符。
<center>Figure 14-13: Link and Lane Number Encoding in TS1/TS2</center>
![](./images/14-13.png)

## 6.2 Designing Devices with Links that can be Merged
设计者一般根据性能与成本的考虑，决定在一条链路上实现多少个通道。PCIe 链路支持多个窄链路聚合成一个更宽的链路，同样的，较宽的链路也可以拆分成多个窄链路。图 14-14 中是一个有一个上游端口和四个 x2 下游端口的 Switch。在本例中，这些通道可以组成两个 x4 的链路。值得注意的是，PCIe 协议要求每个端口必须有作为一个 x1 链路工作的能力。
<center>Figure 14-14: Combining Lanes to From Wider Links(Link Merging)</center>
![](./images/14-14.png)
如图左侧可见，Switch 内部包含 1 个面向上游的逻辑桥以及 4 个面向下游的逻辑桥。事实上，每个端口都需要一个逻辑桥。因此，交换机需要包含 4 个面向下游的逻辑桥，以支持 4 个面向下游的端口。然而，如果交换机如图右方式连接的话，那么剩下的两个逻辑桥就将不工作。在链路训练期间，每个 DSP 中的 LTSSM 将决定具体实现哪些其所支持的连接选项。

## 6.3 Configuration State --Training Examples
### 6.3.1 Introduction
在 Configuration 状态中，链路以及通道的编号工作由 DSP 作为”领导者“发起（比如根节点端口或者交换机面向下游端口）。而终端（Endpoint）以及交换机面向上游端口无法发起这一过程，只能作为”跟随着“响应。接下来将通过一些示例，来使相关概念更容易理解。

### 6.3.2 Link Configuration Example 1
<center>Figure 14-15: Example 1 - Steps 1 and 2</center>
![](./images/14-15.png)
如图 14-15 所示，这个例子中假定双方设备都只支持单个链路，但通道宽度可以是 x4, x2 或者 x1。通道编号分配在设备内部完成，必须从 0 开始连续编号。物理上的通道编号显示在图中所画的设备框内，而逻辑上的，或者说通告的（reported）通道编号，在 TS 有序集上反映。一般来说，逻辑和物理编号是一致的，但也不尽然。

**Link Number Negotiation** 
1. 本例中只支持一条链路，DSP（Downstream Port，向下游发送数据的端口）在所有通道上发送链路编号都为 'N' 的 TS1 有序集，这些 TS1 的通道编号都用 PAD 符号填充。
2. 在配置状态中，USP（Upstream Port）起初发送链路编号和通道编号域都是 PAD 填充的 TS1，但在接收到 DSP 发送的链路编号不为 PAD 符号的 TS1 后，USP 在所有已连接的通道上回复链路编号同样是 'N'，通道编号用 PAD 符号填充的 TS1。基于 USP 的响应，DSP 的 LTSSM 识别到所有四个通道都发出了响应，并使用相同的链路编号 'N'，所以四个通道被配置为同一个链路。通道编号的值 ‘N’ 是一个由具体实现决定的值，这个值不会保存在任何协议定义的寄存器中，并且与端口编号等其他值无关。
<center>Figure 14-16: Example 1 - Steps 3 and 4</center>
![](./images/14-16.png)
**Lane Number Negotiation** 
3. DSP 开始在各个 Lane 上发送 Link 编号相同，但是为各 Lane 各自分配了 0, 1, 2, 3 通道编号的 TS1，如图 14-16 所示。
4. 接收到 Lane 编号不为 PAD 填充符号的 TS1 后，USP 在回复之前首先验证接收到的 Lane 编号和自身物理编号一致。在本例中，DSP 和 USP 的各个通道是正确连接的，编号验证无误，所以 USP 在各通道回复 DSP 的 TS1 中通道了自身的通道编号。当 DSP 接收到编号不为 PAD 符号的 TS1 后，将接收到的通道编号与发送的数值进行比较，如果他们匹配，则这一过程顺利完成。如果不匹配，那么需要采取后续措施，比如部分通道不匹配，而剩余通道匹配，那么链路宽度需要对应调整。如果通道间的连接顺序被颠倒了，那么需要 DSP 支持可选的通道顺序颠倒特性，因为这项特性是可选的，那么有可能会出现通道连接顺序被颠倒，确没有任何一方的设备有能力将其纠正。这是一项重要的电路板设计失误，此时链路配置可能将无法完成。
<center>Figure 14-17: Example 1 - Steps 5 and 6 </center>
![](./images/14-17.png)
**Confirming Link and Lane Numbers** 
5. 因为 DSP 在所有通道上接收到的 Link、Lane 编号和发送数值一致，所以 DSP 开始发送 TS2 序列表示自己已经准备好确认协商结果，并且转移到下一状态 L0，TS2 中的链路编号、通道编号和协商结果一致。
6. USP 在接收链路编号、通道编号和协商结果一致的 TS2 后，也开始发送 TS2 序列表示自己同样做好离开 Configuration 状态，转移到下一个状态 L0，这个过程如图 14-17 所示。
7. 双方端口在收到至少 8 个 TS2 以及至少发送 16 个 TS2 后，发送一些逻辑空闲数据后，转移状态到 L0。

### 6.3.3 Link Configuration Example 2
还是上节中那个有 4 个面向下游通道的设备，不同的是，这个例子中这个设备可以配置为单个 x4，或者配置为两个 x2 链路的组合，或者配置为 4 个 x1 链路的组合，甚至还可以是一个 x2 链路和两个 x1 链路的组合，如图 14-18 所示。

如果 4 个 lane 全部检测到接收端存在，并全部进入 Configuration，那么会有这么几种可能的连接：
- 一个 x4 链路
- 两个 x2 链路
- 一个 x2 链路和两个 x1 链路
- 四个 x1 链路
接下来描述的是协议中决定实施何种配置的一个示例方法。
<center> Figure 14-18: Example 2 - Step 1</center>
![](./images/14-18.png)
**Link Number Negotiation** 
1. DSP 首先在每个 lane 上分配并通告（advertise）单独的 Link 编号。Lane 0 上的 Link 编号是 N，lane 1 上就是 N+1，并以此类推。如图 14-18 所示，图中的 Link 编号只是一个例子，实际上他们可以是不连续的数字。另外，此时 DSP 并不知道和它连接的对端设备情况，在这个过程中 DSP 将尽力去搞清楚每个 lane 可实现的连接方式。
<center>Figure 14-19: Example 2 -  Step 2</center>
![](./images/14-19.png)
2. 在收到 USP 返回的 TS1 序列后（如图 14-19 所示，4 个 lane 分成两组，分别有不同的 Link 编号），DSP 得到两个信息：所有四个 lane 都处于工作状态，并且它们分别连接到了两个 USP，意味当前 DSP 需要分为两个 DSP，每个 DSP 拥有 0、1 两个 lane，如图 14-20 所示。
<center>Figure 14-20: Example 2 - Strps 3,4 and 5</center>
![](./images/14-20.png)
**Lane Number Negotiation** 
3. 各个 Link 上的 Lane 编号会独立进行，不同他们的过程相同：DSP 会在发送的 TS1 中通告其分配给接收 Lane 的 Lane 编号。其中值得注意的是，此时 DSP 在 TS1 的 Link 编号中，对于链路所有的 Lane，只是简单地返回 USP 发送的 Link 编号值。左侧链路两个 Lane 收到的都是 N，右侧链路两个 Lane 收到的都是 N+2。
4. 在本例中，左侧链路的 DSP 和 USP 之间的 Lane 编号相对应，但右侧链路的 Lane 连接则是相反的，DSP 的 Lane0 连到了 USP 的 Lane1，另一条 Lane 也是颠倒连接的。USP 在接收到 TS1 后意识到了 Lane 连接颠倒的现象，如果 USP 支持 Lane 编号颠倒功能的话，它会在内部交换 Lane0 和 Lane1 的编号，从而返回给 DSP 的 TS1 中保持原本 DSP 提议的 Lane 编号不变，如图 14-20 所示。如果 USP 并不支持这项特性，那么它会在返回的 TS1 中通告自己的物理 Lane 编号情况（和 DSP 提议的 Lane 编号相反），这样一来 DSP 也能意识到这个问题，并有机会在 DSP 内部颠倒 Lane 的逻辑编号来解决这个问题。
5. Lane 顺序颠倒是一项双方端口可选的特性。如果 USP 检测到 Lane 编号颠倒，并有能力颠倒自己 Lane 的逻辑编号，那么 USP 会在内部颠倒自己 Lane 的逻辑编号，并返回 Lane 编号不变的 TS1。这样一来，DSP 都不会意识到这个问题的存在。如果 USP 不支持这项功能，那么 DSP 会发现其接收到的 TS1 的 Lane 编号相比它发送的 TS1 是颠倒的。如果 DSP 支持这项特性，那么它会纠正自身 Lane 逻辑编号的顺序，并在 TS2 中发送纠正后的 Lane 编号做为确认。
6. DSP 在接收到 Link 编号和 Lane 编号与所通告的编号一致的 TS1 后，各个 DSP 端口独立开始发送 TS2，作为它们已经准备好使用协商值，并做好切换至 L0 状态。
7. USP 在收到 Link 和 Lane 编号没有变化的 TS2 后，也可以发送相同的 TS2 序列。
8. 双方端口在收到至少 8 个 TS2 以及至少发送 16 个 TS2 后，会发送一些逻辑空闲数据，然后转移状态到 L0。

### 6.3.4 Link Configuration Example 3: Failed Lane

最后讨论在配置的过程中，某个 lane 如果不能正常工作会发生什么。这个例子基于 USP 的 lane2 不能正常工作，如图 14-21 所示。需要注意的是这个 lane 没有严重到要断开物理连接，如果这么严重， DSP 将不会把它识别为接收端，并不会把它纳入链路。然而，即使 lane 的物理连接性没有问题，lane2 的接收端或者发送端将无法完成配置工作（可能两端都无法完成）。

在本例中，链路训练过程可能会显著变长，因为在大多数状态跳转过程中，都需要等待所有 lane 就绪，如果只是一部分 lane 就绪，剩下一些没有，那么就需要等待超时条件满足才能跳转状态。

接下来的步骤描述了 Configuration 状态机各个子状态的跳转，其处理这种异常情况的过程。

<center> Figure 14-21: Example 3 - Steps 1 and 2 </center>
![](./images/14-21.png)

**Link Number Negotiation** 
1. 虽然 USP lane2 接收端存在问题，但 DSP 在进入 Configuration 状态后还是按照流程行事。DSP 在所有 lane 上发送 Link 编号为 'N'、lane 编号域填充 PAD 的 TS1 序列。
2. lane0、1、3 收到了 DSP 发送 Link 编号域为非填充值得 TS1，所以按照协议回复 DSP 相同的 TS1。然而，lane2 因为接收故障，没能收到 DSP 发送的 TS1，所以 lane2 发送端继续发送 Link 编号、Lane 编号都为填充字符的 TS1，如图 14-21 所示。
**Lane Number Negotiation** 
3. DSP 在收到 lane0、1、3 发送的带有相同链路编号的 TS1 后，开始等待 lane2 也完成同样的工作，直至超出协议规定的最大超时时间。如果此时 lane2 还是没有收到消息，那么 DSP 就会意识到这个链路只能被训练为 x2 宽度的链路。此时 DSP 会向 USP 通告给 lane0 和 lane1 分配的 lane 编号，而 lane2、lane3 上只发送 Link 编号和 Lane 编号都使用填充字符 PAD 的 TS1。
4. 当 USP 在 lane0、lane1 上收到有有效 lane 编号的 TS1，而 lane3 又变回了填充字符 PAD 时，USP 回复相同的 lane 编号，而其他 lane 开始（指 lane3，lane2 是继续发送）发送 Link、Lane 编号都使用填充字符 PAD 的 TS1，如图 14-22 所示。
<center>Figure 14-22: Example 3 - Steps 3 and 4 </center>
![](./images/14-22.png)
**Confirming Link and Lane Number**
5. 因为 lane0、lane 1 上收到的 TS1 的 Link、Lane 编号都与预期匹配，所以 DSP 准备结束此协商并开始发送具有相同 Link、Lane 编号的 TS2，并准备进入 L0 状态。而在其他的 lane 上，继续发送 Link、Lane 编号都是用填充字符 PAD 的 TS1。
6. USP 在收到 Link 编号、Lane 编号和协商结果一致的 TS2 后，也开始发送 TS2 序列表示同样做好离开 Configuration 状态，转移到下一个状态 L0。而在其他 Lane 上继续发送 Link、Lane 编号使用填充字符的 TS1，这个过程如图 14-23 所示。
<center>Figure 14-23: Example 3 - Steps 5 and 6</center>
![](./images/14-23.png)
7. 在双方端口收到至少 8 个 TS2 以及至少发送 16 个 TS2 后，会发送一些逻辑空闲数据，然后转移状态到 L0。其他未完成配置的通道，如本例中的 Lane2、Lane3，转为电气空闲状态，直到下一次 DSP 发起链路训练操作，这时会继续尝试正常的训练过程，以期故障排除并完成配置。
## 6.4 Detailed Configuration Substates
本小节将详细讨论 Configuration 各子状态，其图如 14-24 所示。
<center>Figure 14-24: Configuration State Machine</center>
![](./images/14-24.png)

### 6.4.1 Configuration.Linkwidth.Start
有两种情况会进入该状态，一种是 Polling 状态正常结束跳转，另一种是在 Recovery 状态中发现 Link 或 Lane 编号相较于之前发生了变化，因此无法正常完成 Recovery  状态的跳转。

**A. Downstream Lanes** 
**During Configuration.Linkwidth.Start**  
DSP 作为链路的发起者，将在所有工作的 lanes 上发送 Link 编号不为 PAD 的 TS1 序列（仅在 LinkUp 标志尚未置起，以及链路宽度得配置尚未进行时）。此时 TS1 中 Link 编号不为 PAD，而 Lane 编号仍为 PAD。协议对 Link 编号具体数值得唯一规定：如果设备支持多 Links，那么分配给不同 Link 的编号不能相同。Link 编号的数值是两个链路伙伴之间的本地值，软件无需追踪所有链路的编号数值，也没有必要使整个系统中的所有链路编号保持不同。

如果 `upconfigure_capable` 标志被置为 1b，那么也会在不工作的 lane 上发送 TS1，只要这些 lane 曾经收到过两个连续的 Link、Lane 编号都为 PAD 的 TS1。
- 从 Polling 状态进入此子状态时，所有检测到接收端的 Lane 会被视为工作通道（Active Lane）
- 从 Recovery 状态进入此子状态时，所有历经 `Configuration.Complete` 状态的 Link 上的 Lane，都会被视作工作通道（Active Lane）
- DSP 必须在发送的 TS1 中通告自身所支持的所有速率，即使该端口不打算使用的速率。

**Crosslinks 交叉链路** 
对于支持交叉链路特性的 DSP 来说，当 Linkup = 0b 时，需要在所有检测到接收端的 Lane 上至少发送 16-32 个 TS1，其中 Link 编号非 PAD，而 Lane 编号是 PAD。之后，DSP 将根据接收到的内容判断对端是否存在交叉链路。

**Upconfiguring the Link Width 恢复链路宽度**
在 LinkUp = 1b 时，如果 LTSSM 想要恢复较大的链路宽度，那么会在所有当前工作的 Lane、想要激活的非工作 Lane 以及接收到 TS1 的 Lane 上，发送 Link 编号和 Lane 编号使用 PAD 填充的 TS1。当 DSP 在某个 Lane 上接收到两个连续的 TS1 回复后，延时 1ms，然后发送指定 Link 编号的 TS1。
- 在激活不工作的 Lane 时，发送端必须等待发送共模电压（Tx common mode voltage）稳定，才能退出电气空闲状态并开始发送 TS1。
- 对于组合为同一链路的 Lane，他们的 Link 编号必须相同。只能在支持多链路配置的不同链路之间分配不同的链路编号。

`Exit to "After a 24ms timeout if none of the other confitions are true"` 
任何收到过至少一个 Link、Lane 编号均为 PAD 的 TS1 

### 6.4.2 Configuration.Linkwidth.Accept
### 6.4.3 Configuration.Lanenum.Wait
### 6.4.4 Configuration.Lanenum.Accept
### 6.4.5 Configuration.Complete
### 6.4.6 Configuration.Idle
# 7. L0 State
## 7.1 Speed Change
## 7.2 Link Width Change
## 7.3 Link Partner Initiated
# 8. Recovery State
## 8.1 Reasons for Entering Recovery State
## 8.2 Initiating the Recovery Process
## 8.3 Detailed Recovery Substates
## 8.4 Speed Change Example
## 8.5 Link Equalization Overview
## 8.6 Detailed Equalization Substates
### 8.6.1 Recovery.Equalization
### 8.6.2 Recovery.Speed
### 8.6.3 Recovery.RcvrCfg
### 8.6.4 Recovery.Idle
## 8.7 L0s State
## 8.8 L1 State
## 8.9 L2 State
## 8.10 Hot Reset State
## 8.11 Disable State
## 8.12 Loopback State
### 8.12.1 Loopback.Entry
### 8.12.2 Loopback.Active
### 8.12.3 Loopback.Exit
# 9. Dynamic Bandwidth Changes
## 9.1 Dynamic Link Speed Changes
## 9.2 Upstream Port Initiates Speed Change
## 9.3 Speed Change Example
## 9.4 Software Control of Speed Changes
## 9.5 Dynamic Link Width Changes
## 9.6 Link Width Change Example
# 10. Related Configuration Registers
## 10.1 Link Capabilities Register.
### 10.1.1 Max Link Speed [3:0]
### 10.1.2 Maximum Link Width[9:4]
## 10.2 Link Capabilities 2 Register
## 10.3 Link Status Register
### 10.3.1 Current Link Speed[3:0]
### 10.3.2 Negotiated Link Width[9:4]
### 10.3.3 Undefined[10]
### 10.3.4 Link Training[11]
## 10.4 Link Control Register
### 10.4.1 Link Disable
### 10.4.2 Retrain Link
### 10.4.3 Extended Synch