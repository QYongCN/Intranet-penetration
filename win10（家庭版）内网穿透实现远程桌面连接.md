# win10（家庭版）内网穿透实现远程桌面连接

## 一、前清提要

​		今天，在经过多次连接失败后，Teamviewer声称检测到用于商业用途。于是，比较流畅的teamviewer不能用了。免费的向日葵太卡，付费是不可能付费的。想起内网连接服务器时远程桌面的流畅，于是乎有了这篇教程。

​		对Win主机进行远程控制，最好的工具自然是系统自带的**“远程桌面连接”**功能，可以直接进行点对点的桌面连接，被控主机的剪贴板和控制主机无缝衔接，稳定性和延迟也比第三方软件不知道高到哪里去了。然而脱离了局域网的环境，远程桌面连接会遇到一些阻碍。

<img src="C:\Users\乔勇\AppData\Roaming\Typora\typora-user-images\image-20200815181044788.png" alt="image-20200815181044788" style="zoom:50%;" />

​		远程桌面最重要的一点就是被控端电脑的IP得是公网IP，而校园网、移动宽带、局域网等分配的IP都是内网IP，包括电信和联通的宽带默认分配的也是内网IP，并且由于现在IPv4地址紧张，有些地方电信和联通的宽带都不一定要的到公网IP。连在这些网络中，虽然能正常上网，但你被分配到的IP是内网IP，远程桌面当然连不上了。

​		针对以上情况，唯一可能的只有内网穿透，免费内网穿透也有不少，但要么限制流量，要么网速感人，选来选去，最终在无意间发现了这个内网穿透服务——SAKURA FRP内网穿透。这是一个公益项目，相当良心，（如果经常使用或者有钱的话可以给作者赞助一下，毕竟开发这个确实不容易且非常良心），免费用户不限流量，虽然限速，但每天可以通过签到获得高速流量，每次签到最少获得1G，新注册用户送5G高速流量，限速后依然有4Mbps！4Mbps呐！很多收费的内网穿透VIP用户都没这个网速，研究内网穿透这么久来，没见过比这更良心的了！

**注意：以下步骤都是在远程主机上完成，因此需要使用Teamviewer/Anydesk/向日葵进行远程操作。如果被控主机使用专业版Windows，则可以省略步骤1。之所以使用Sakura Frp是因为这是一项免费服务。**

## 二、解决方案

### step.1 安装**[RDP Warp](https://link.zhihu.com/?target=https%3A//github.com/stascorp/rdpwrap)**恢复Win10家庭版系统的远程桌面功能

​		RDP Warp是一个良心项目，其作用就是恢复历代Windows家庭版系统被阉割的远程桌面服务端功能。如果你是缺省的家庭版系统，那么是肯定不可以作为远程桌面的受控方的（但是可以连接别人），因此需要安装这么个小玩意。

​		**下载地址：https://pan.baidu.com/s/1xStO71MbT-wHfVZBwQhsiQ       提取码：bbep**

![image-20200815182953147](C:\Users\乔勇\AppData\Roaming\Typora\typora-user-images\image-20200815182953147.png)

​		首先运行RDPCheck.exe，这是一个测试程序，会尝试建立一个和本机之间的远程连接。如果连接成功，那么说明你的主机是支持完整的远程桌面连接功能的，也就不需要进行后续操作了。如果失败，那么接着以管理员身份运行install.bat，执行安装。安装程序结束之后，运行RDPConf.exe，查看目前的远程桌面服务的运行状态。

​		![image-20200815183753535](C:\Users\乔勇\AppData\Roaming\Typora\typora-user-images\image-20200815183753535.png)

​		如果你的页面如上图所示，Diagnostics里面的三个状态都是绿的，那么远程桌面连接已经没有问题了，不需要进行后续操作。**这里可以对远程连接服务的本地端口进行设置，默认和专业版系统一致，都是3389，这个端口在后续配置隧道的时候会用上。**

​		**如果你的Listener state一项是红色的，显示not listening [not supported]，**那么很遗憾，你遇到了和我一样的问题。我找到了正确的解决方法，以下链接里有描述：https://github.com/stascorp/rdpwrap/issues/889

​		需要做的是：

- 进入管理员权限的Powershell/CMD
- **运行命令net stop termservice**，关闭Remote Desktop Services服务
- 把下载得到的**rdpwrap.ini文件替换到**C:\Program Files\RDP Wrapper文件夹内（https://github.com/rustaman2012/rdpwrapper.git）
- 运行命令net start termservice，恢复Remote Desktop Services服务

​       我没有重启，毕竟是在远程操作，怕网络断了就连不回去了，实践证明并不影响效果。操作完成之后，再次打开RDPConf.exe查看运行状态，都是绿的了。再运行RDPCheck.exe，发现连接成功，建立了一个和本机之间的远程桌面。

​		至此，家庭版系统上的远程桌面连接服务端功能已经恢复，接下来还需要进行内网穿透。

### step. 2 使用Sakura frp进行内网穿透

#### 1、Sakura frp账号注册

​		首先注册Sakura frp账号，百度Sakura frp，一般第一个就是官网，官网长这样：

<img src="C:\Users\乔勇\AppData\Roaming\Typora\typora-user-images\image-20200815184611325.png" alt="image-20200815184611325" style="zoom:50%;" />

​		点击注册账号，填写资料注册账号。注册完成后的界面长这样，左边是菜单栏。记住自己的访问密钥，待会用得上，同时不要把访问密钥透露给其他人。

<img src="C:\Users\乔勇\AppData\Roaming\Typora\typora-user-images\image-20200815184855572.png" alt="image-20200815184855572" style="zoom:50%;" />

​		点击左边菜单栏的软件下载，根据系统类型选择对应的客户端进行下载安装，win10选第一个就好了。下载，然后安装。

<img src="C:\Users\乔勇\AppData\Roaming\Typora\typora-user-images\image-20200815185018171.png" alt="image-20200815185018171" style="zoom:50%;" />

​		至此，网页注册完成，注册可以在任意电脑完成，但下载客户端一定要下到被控电脑上。

#### 2、允许连接远程桌面

右击此电脑，点击左边的远程设置。点击上面的远程，上面把“允许远程协助连接这台计算机”前面的框勾上，下面选择“允许远程连接到此计算机。

<img src="C:\Users\乔勇\AppData\Roaming\Typora\typora-user-images\image-20200815185403297.png" alt="image-20200815185403297" style="zoom:50%;" />

#### 3、内网穿透

​		打开刚才下载的软件，长这样，用刚才网页上的访问密钥进行登陆。

<img src="C:\Users\乔勇\Desktop\批注 2020-08-15 185719.png" alt="批注 2020-08-15 185719" style="zoom:50%;" />

​		登陆完成后，点左边菜单栏的隧道，可以看到目前是空的。

​		点击加号新建隧道，输入本机IP。输入本机IP，端口填远程桌面的端口，默认是3389，隧道名称随便填，如果不填的话就会产生一个随机的名称，隧道类型选择TCP，远程端口可以自己指定，范围是10240~65535，不能和已有的重复，如果你连远程端口是啥都不知道就直接默认0吧，会给你随机分配一个的。服务器看情况自己选择，一般没有特殊需求就选国内的服务器，关于服务器的详细情况可以去网站上看。如果你是小白的话注意看一下左边的监听端口，类型为TCP的有没有3389这个端口，一般就在前面，如果没有的话就是远程桌面端口没有开启。

<img src="C:\Users\乔勇\AppData\Roaming\Typora\typora-user-images\image-20200815190138110.png" alt="image-20200815190138110" style="zoom:50%;" />

​		点击创建，会弹出创建成功提示框，点击是。可以看到多出一个隧道，就是刚才创建的隧道。

<img src="C:\Users\乔勇\Desktop\批注 2020-08-15 190410.png" alt="批注 2020-08-15 190410" style="zoom:50%;" />

​		点击开启刚才的隧道，会弹出日志信息，记住这个日志信息上面的IP或者服务器域名，待会通过这个IP连接被控电脑。

​		至此，内网穿透全部搞定，被控电脑全部设置完毕，接下来是主控电脑的设置。

#### 4、远程桌面连接

​		此步在主控电脑上完成，打开远程桌面连接，路径为Windows->附件->远程桌面连接。

<img src="C:\Users\乔勇\AppData\Roaming\Typora\typora-user-images\image-20200815191556168.png" alt="image-20200815191556168" style="zoom: 67%;" />

​		输入刚才被控电脑的IP，点击连接。至此，远程桌面连接全部完成，尽情愉快的使用吧！