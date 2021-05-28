
> 本文针对Android 4.2后google开发的蓝牙栈 BlueDroid中存在的粗粒度权限管理问题，提出了并实现了在多版本中的攻击**BadBluetooth**。
>
> 通过将蓝牙设备伪装为键盘，网络接入点和耳机，同时配合Android 恶意app发起静默配对，最终实现控制手机截屏偷取用户隐私数据，劫持通信流量，甚至在锁屏状态下拨打电话等攻击。
>
> 最后，作者在AOSP项目上实现了对应的防御框架



## Introduction

本文从逻辑层面对蓝牙进行了系统的研究，包括攻击者模型、设备认证、授权、安全策略等底层假设。

虽然各类OS都存在蓝牙模块，但考虑到Android系统的普及性，所以主要研究了Android系统蓝牙模块存在的问题。



**贡献**：

- 发现了几个Android系统在蓝牙设计和实现中的漏洞，包括设备配置文件修改，粗粒度的认证和授权机制等
- 通过这些漏洞，能够在真实环境中实现攻击，造成信息泄露等威胁
- 实现了针对这一问题的防御框架并进行了效果评估



## BackGround

背景介绍中主要介绍了蓝牙的相关知识，之前我也没接触过，所以认真看了下

### Bluetooth Stack

**蓝牙协议栈的结构**如下图：

<img src="https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2020-07-11-134146.jpg" alt="image-20200519133616474" style="zoom:50%;" />

蓝牙栈是一个多层的结构，包括物理层、链路层、中间件层和应用层。下层由蓝牙芯片实现，包含无线控制器、系带控制器等。它们通过主机控制器接口(Host Controller Interface)与操作系统进行通信，中间件层的协议由操作系统实现。

中间件层的基础层协议是逻辑链路控制适配协议(L2CAP)，它管理两个蓝牙设备之间的连接，实现了QoS、流控、分片和重装机制等功能。在L2CAP的基础上，设计了一系列面向应用的协议。（RFCOMM,SDP等）

### Bluetooth Profile

蓝牙配置文件是为了规范不同厂商设备间的通信。在配置文件中包含了引导通信的设置，例如格式、协议等，目前共有30多种标准配置文件。

最常用的配置文件是耳机配置文件(Headset Profile，HSP)，它规定了蓝牙耳机如何与手机通信。



### Bluetooth Connetion

**蓝牙的连接过程**：

- **发现阶段**：扫描发现附近设备，包括设备名字，设备种类，设备profile
- **配对阶段**：致辞多种配对模式，一般需要用户输入pin码或者比较数据
- **建立连接**：两个设备配对后共享link key，用于加密双方通信的数据



### Android Bluetooth

之前的Android的是linux的BlueZ栈，但从Android 4.2开始，google实现了自己的蓝牙栈BlueDroid

**BlueDroid中的权限管理**：

- normal-level：无需用户确认，用来请求和接收连接
  - BLUETOOTH
  - BLUETOOTH_ADMIN
- dangerous-level，：需要用户授权，扫描附近设备，用来获取用户位置
  - ACCESS_COARSE_LOCATION
  - ACCESS_FINE_LOCATION
- signature-level：需要用户授权，用户需要交互的配对过程
  - BLUETOOTH_PRIVILEGED 



## Design Weaknesses

在BlueDroid的设计中，主要存在以下**五个weakness**

- **Weakness #1: Inconsistent Authentication Process on Pro- files.**

  - 在配对过程中，配置文件不会被列出。如果在配对后对配置文件进行修改，配对仍然会成立

  - 如果连接时为耳机的配置文件，在连接后修改为输入设备的配置文件，那么就能够通过蓝牙进行输入了

    <img src="https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2020-07-11-134154.jpg" alt="image-20200519143519758" style="zoom:67%;" />

- **Weakness #2: Overly Openness to Profile Connection.**

  - 一旦连接建立，主机就会尽力连接到远程设备声称的所有配置文件，而不会向用户解释风险，也不会让用户审核这些连接。即使用户稍后可以在设备详情菜单中断开某些配置文件的连接，但主机不会记住这样的决定。当下次设备配对时，连接将被重新建立。

- **Weakness #3: Deceivable and Vague UI.**

  - 用户浏览配对的蓝牙设备列表时，能够看到名称和图标，但这是能够伪造的
  - 恶意设备能够修改名称，通过改变CoD(Class of Device)号改变现实的图标
  - 缺少UI提示蓝牙相关信息。例如，只有两个事件会在通知栏中提示：显示蓝牙已打开，显示已连接远程设备。

- **Weakness #4: Silent Pairing with Device.**

  - 当从设备端发送配对请求时，Android系统会弹出对话框让用户确认。但是，如果由手机发起连接，则可能没有通知。 比如，当设备没有显示能力或输入能力（例如，耳机）

- **Weakness #5: No Permission Management for Profile. **

  - Android通过权限限制应用程序是否可以访问蓝牙设备，但是权限管理太过粗糙
  - 例如使用BLUETOOTH_ADMIN权限能访问配置文件，虽然在新版中受到了限制，但能通过java反射机制实现



## ATTACK OVERVIEW

**攻击者模型**：

- 手机上安装具有Bluetooth权限的恶意app-BLUETOOTH和BLUETOOTH_ADMIN是一般权限 - 无需请求用户同意权限申请 
- Bluetooth设备是受攻击者控制的 - XcodeGhost攻击 - 通过设备其他漏洞获得设备权限后插入恶意代码



**攻击流程**：

<img src="https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2020-07-11-134202.jpg" alt="image-20200519144807796" style="zoom:67%;" />

- **修改配置文件(#1 #2 #3)**：配对完成后，设备添加其他配置文件，并在攻击完成后删除
- **静默连接(#4)**：使用静默方式连接恶意蓝牙设备
- **使用敏感的配置文件(#5)**：通过java反射机制操作敏感的配置文件



**攻击步骤**：

- 启动恶意app，并保持后台运行，直到监测到手机屏幕关闭时开始攻击
- 通过调用BluetoothAdapter.enable静默配对已知地址的恶意设备
- 设备等待从app发来的命令，命令通过蓝牙信道传送，或通过网络转发
- 接到命令后，设备使用敏感的配置文件，App利用存在的配置文件功能
- 设备恢复正常的状态，App使用removeBond取消配对，以免引起注意



## Attack

作者根据现有的android profile，总结并实现了攻击，其中HID、PAN和HFP/HSP这三个profile可以被攻击者利用

<img src="https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2020-07-11-134207.jpg" alt="image-20200519145350562" style="zoom:67%;" />

### HIP(Human Interface Device)

例如键盘和鼠标，当HIP接入后，就能向android手机输入内容了

<img src="https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2020-07-11-134212.jpg" alt="image-20200519151056154" style="zoom:50%;" />



**攻击策略**：

- 自适应攻击：主要在于识别鼠标位置等，通过手机手机的信息
- 输入：通过模拟按键和鼠标点击构造输入
- 输出：截屏或者选择文字赋值粘贴进行输出



**危害**：

- 信息窃取
- 操控系统和App
- 盗取密码等敏感内容



### PAN(Persinal Area Networking)

<img src="https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2020-07-11-134216.jpg" alt="image-20200519151935464" style="zoom:67%;" />

**危害**：

- 网络嗅探和欺骗：由于手机能够通过蓝牙访问互联网，因此可以执行中间人攻击，拦截流量
- 偷网络流量：通过蓝牙共享手机网络



### HF(Hands Free)

<img src="https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2020-07-11-134221.jpg" alt="image-20200519153546573" style="zoom:67%;" />

**危害**：

- 控制电话，拨打任意号码
- 语音命令控制

### Other Profiles

除了上述三种profile的攻击，也有一些其他的攻击可以实现，但他们会通知用户批准请求，因此并不隐秘



## IMPLEMENTATIONS AND EVALUATIONS

设备：

- 树莓派2代（Linux OS）
- CSR8510 USB蓝牙适配器
- Google Pixel 2 （Android 8.1）



实现：

Raspberry Pi 2 + 1100 行Python 代码（PyBluez）

- HID attack，raw L2CAP
- PAN attack，tcpdump和dnsmasq
- HFP attack，pulseaudio和ofono

<img src="https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2020-07-11-134226.jpg" alt="image-20200519153858880" style="zoom:50%;" />

在测试中，Android5.0到8.1之间的测试版本都攻击成功了



## PROFILE BINDING FOR ANDROID

为了解决发现的问题，作者也提出并实现了相应的防御框架，对profile进行控制

<img src="https://ycdxsb-1257345996.cos.ap-beijing.myqcloud.com/blog/2020-07-11-134238.jpg" alt="image-20200519154141699" style="zoom: 50%;" />

经过检验，这个上层防御框架能够在比正常使用多12%的时间，实现很好的防御



## 个人感觉

文中主要对安卓蓝牙协议栈进行了研究，但其实文中提出的weakness中，有的在其他系统中也存在，比如静默匹配，以及提示弹窗问题在大部分系统中都有，可能可以对这一方法进行一些系统的扩展和进一步研究。

同时，由于蓝牙是基于连接的问题，可以想到常用的wifi，苹果的隔空投送等也可能有问题。

例如，刚看论文的时候想到了一个wifi重连的问题，虽然没仔细研究，但可能可以根据wifi自动重连的机制，伪造wifi，截取用户的流量，形成中间人攻击



