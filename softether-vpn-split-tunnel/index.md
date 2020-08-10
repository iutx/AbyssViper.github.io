# Softether-VPN 拆分隧道


### 需求来源

​	SoftEther VPN介绍[移步此处](https://zh.wikipedia.org/zh-cn/SoftEther_VPN)

​	对于VPN来说，存在**远程网关与本地网关**的概念，如下图以SoftEther VPN 的 SecureNAT 配置为例，接入VPN后本地路由表的对比 。



![](https://i.loli.net/2020/02/26/aIHLx475TWp3vlw.png)

​	可以很明显的看出，如果使用远程网关，所有的**流量都走VPN隧道**，这样不仅给VPN服务器造成很大的压力，而且大部分日常的网络访问都需要去VPN服务器绕一圈，显然这样是很不合理的。

​	所以比较好的方案是**除了指定的地址段**走VPN隧道，其余流量全部走本地的默认网关。

​	显然使用本地网关是无法直接访问到异地的内网地址的，缺少了一步静态路由。比如使用本地网关的情况下，异地存在一个web应用，访问地址 https://192.168.7.5, 连接VPN后，我tracert一下，如图所示，经过几跳以后，在公网直接超时了。

![](https://i.loli.net/2020/02/26/Jzeh1dVi2LFRaCb.png)

​    对于已经接入VPN的本机来说，192.168.200.1 就是通过虚拟局域网前往异地内网的网关，所以如果我们增加这么一条静态路由，指向如果走7网段直接通过网关192.168.200.1，添加后再次tracert 一下, 可以看到直接通过远程网关访问到了异地内网的机器

```bat
route add 192.168.7.0 mask 255.255.255.0 192.168.200.1
```

![](https://i.loli.net/2020/02/26/1HuA4CXDqnbNriB.png)

​    之前在同样使用SoftEther VPN的项目上，解决该问题的方法是每次使用运行一下上述命令的bat脚本，该解决方法每次接入VPN之前都需要运行一下，如果加了 -p 参数添加为永久路由，又会在某些网络环境下造成不可避免的冲突。





### Split Tunneling

   Split Tunneling （拆分隧道），是SoftEther-VPN中比较强悍的一个功能。具体位置在SecureNAT配置界面就可以找到，简单来讲 拆分隧道可以理解为推送静态路由，接入 VPN 以后，server端会推送设置的静态路由到client端，断开VPN后，推送的静态路由失效，完美的解决了上述问题带来的痛点。

![](https://i.loli.net/2020/02/26/CqKDUlWzROFdYZL.png)



   现实总是这样一次次的打击，提示该功能并不适合开源版本，其实根本原因也并不是开源版本不能使用该功能，我再海外的VPS中用同样的包进行了部署测试，该功能是没有问题的，从网上查到的信息，基本意思就是天朝跟岛国不可以使用该功能在内的一部分功能(当然仅限于官方下载的编译好的版本，对于自己进行源码编译是不限制的)

![](https://i.loli.net/2020/02/26/SyAj8n7d4sHw2YK.png)

  

### 解除限制

#### 下载源码

- [下载地址](https://www.softether-download.com/?product=softether)， 组件选择 **Source Code of SoftEther VPN**

- 如果是生产环境在用，建议下载在用版本的源码



#### 删除限制部分代码

- 解压后，在以下路径中找到Server.c文件，编辑 

```shell
/your_tar_path/src/Cedar/Server.c
```

- 可以看出Server端代码在以下 两个函数中 出现了关键词 CN 与 JP

```c
void SiGetCurrentRegion(CEDAR *c, char *region, UINT region_size)
bool SiIsEnterpriseFunctionsRestrictedOnOpenSource(CEDAR *c)
```

- SiIsEnterpriseFunctionsRestrictedOnOpenSource函数中调用了SiGetCurrentRegion函数，最终的逻辑判断还是发生在SiIsEnterpriseFunctionsRestrictedOnOpenSource函数该段代码

```c
if (StrCmpi(region, "JP") == 0 || StrCmpi(region, "CN") == 0)
{
	ret = true;
}
```

- 我们直接把 ret 的赋值改为  false；当然更改方法多种多样

#### 编译

```shell
yum -y groupinstall "Development Tools"
yum -y install readline-devel ncurses-devel openssl-devel
./configure
make
```



#### 部署

- 编译完成后，会在如下路径生成vpnserver 以及hamcore.se2文件

  ```shell
  bin/vpnserver/
  ```

- 直接用上述两个文件替换掉原部署的vpnserver以及hamcore.se2即可

- 注意: 替换前注意备份原目录，替换前注意停止vpnserver服务



#### 测试

- 再次Manager连接进行路由推送，并没有弹出窗口限制
- 客户端拨入VPN，再次查看路由表，发现路由已经推送过来了；断开VPN后该条路由被清理

![](https://i.loli.net/2020/02/27/OKMf1gVhsnZ9lvH.png)