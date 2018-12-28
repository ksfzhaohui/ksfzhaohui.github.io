**国内外vps主机提供商所提供的主机大多是基于Xen、OpenVZ、KVM、Hyper-V、VMWare五种虚拟化技术。**  
  
**1.Xen**  
Xen 由剑桥大学开发，它是基于硬件的完全分割，物理上有多少的资源就只能分配多少资源，因此很难超售。可分为Xen-PV（半虚拟化），和Xen-HVM（全虚拟化）。  
Xen-PV：半虚拟化，所以它仅仅适用于 linux 系列VPS，但它的性能损失比较少，大概相对于母机的4%-8%左右。  
Xen-HVM：全虚拟化，可以安装windows或自由挂载ISO文件安装任意系统，由于是全虚拟化，所以性能损失较大，大概相对于母机性能损失8%-20%左右。

优点：在资源有限的情况下，基本无法超售，但是市面上很多 VPS 商家采用超大内存的服务器进行销售 Xen VPS，也就是所谓的变相超售。  
缺点：相对于母机性能损失比较大

Xen可用系统：Xen-PV：纯Linux，Xen-HVM：支持Windows、Linux等。  
Xen代表商家：[Linode.com](http://linode.com/)

**2.OpenVZ**  
OpenVZ（简 称OVZ）采用SWsoft的Virutozzo虚拟化服务器软件产品的内核，是基于Linux平台的操作系统级服务器虚拟化架构。这个架构直接调用母服务器（母机）中的内核，模拟生成出子服务器（VPS，小机），所以，它经过虚拟化后相对于母服务器，性能损失大概只有的1-3%。

优点：采用半虚拟化技术，VPS 的所有文件均放置于所在的服务器上，在同样的性能测试下，OpenVZ 会比 Xen 占一定优势  
缺点：OpenVZ可以超售，意思味着一台服务器总共16G内存，他可以开出配置为1G内存×17台以上的子服务器。因为他的虚拟架构关系属于：客户用多少，就扣除母服务器多少，所以OpenVZ架构的VPS较为便宜。但由于存在超售因素，如果服务商毫无休止的超售会导致服务器的性能急剧下降。

OpenVZ可用系统：Linux（不支持Windows）  
OpenVZ代表商家：[Buyvm.net](http://buyvm.net/)、[bandwagonhost.com](https://bandwagonhost.com/)

**3.KVM**  
KVM是Linux下的全功能虚拟化架构，基于KVM架构的VPS，默认是没有系统的，可自己上传ISO或调用服务商自带的ISO手动安装系统。

优点：可以自己载入 iso 进行系统安装等操作，更适合喜欢 DIY 的用户  
缺点：由于KVM架构全功能虚拟化架构，甚至拥有独立的BIOS控制，所以对母服务器性能影响较大，所以基于KVM的VPS较贵；另外一点是技术不是很成熟

KVM可用系统：Windows、Linux系列  
KVM代表商家：[hostigation.com](https://hostigation.com/)

**4.Hyper-V**  
Hyper-V是微软的一款虚拟化产品，大部分国内的VPS服务商使用这个架构，主要是因为其转为Windows定制，管理起来较为方便。目前的Hyper-V也支持Linux，只不过性能损失比较严重。  
Hyper-V目前不能超售内存，但可超售硬盘，硬盘是根据客户使用情况扣除。

优点：采用微软内核构架，兼顾安全和性能的要求  
缺点：只支持Windows和Linux操作系统，内存管理模式限制了一台物理主机上加载虚拟机的数量，不支持silent模式来一次性移植多个虚拟主机。

Hyper-V可用系统：Windows、Linux

**5.VMWare**  
VMWare 是全球桌面到数据中心虚拟化解决方案的领导厂商开发的一款全功能完全虚拟化的软件。但由于VMWare用于开设类似VPS（含独立面板）的系列产品授权费用非常昂贵，所以大部分使用VMWare服务商会使用 VMware工作站（VMware Workstation）提供VPS。

使用VMware工作站（VMware Workstation）开设的VPS是无控制面板的，操作系统需要服务商手动安装，但现在网上寻找VMware Workstation的神KEY非常容易，对于VPS服务商来说节省不少成本。一般用于新创业的VPS服务商。

使用VMWare Workstation实质上的VPS可以超售，因为其和OpenVZ架构一样，子机用多少内存，就扣除系统多少内存，但如果物理内存不足时可能导致母服务器使用Windows虚拟内存。

优点：对多种操作系统的支持比较好  
缺点：Vmware的磁盘I/O性能一直表现不好。另外，CPU性能也比不上Hyper-V

VMWare可用系统：Windows、Linux系列

**如何判断vps是使用哪种虚拟技术**  
1.命令ifconfig：查看网卡，openvz的一般都是venet0:* ，xen、kvm的一般都是eth*  
2.命令ls /proc/：一般Xen的VPS，/proc目录下面会有xen的目录，openvz的会有vz目录  
3.通过专门的软件：[virt-what](http://people.redhat.com/~rjones/virt-what/)  
可以执行如下命令安装(需要安装好gcc、make)：

```java
wget http://people.redhat.com/~rjones/virt-what/files/virt-what-1.15.tar.gz
tar zxvf virt-what-1.15.tar.gz
cd virt-what-1.15/
./configure
make && make install
```

在centos可以直接执行：

```cs
yum install virt-what
```