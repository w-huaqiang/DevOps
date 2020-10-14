1. `kernel: XFS: possible memory allocation deadlock in kmem_alloc(mode: 0x250)`

```bash
# 内存死锁,可以使用以下方式尝试解决
echo 1 > /proc/sys/vm/drop_caches  #一般无法解决，需要升级内核
```

```bash
# 在 CentOS 7.× 上启用 ELRepo 仓库
$ rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
$ rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm

# 列出可用的系统内核相关包:
$ yum --disablerepo="*" --enablerepo="elrepo-kernel" list available

# 安装新内核
$ yum --enablerepo=elrepo-kernel install kernel-ml

# 为了让新安装的内核成为默认启动选项，你需要如下修改 GRUB 配置,打开并编辑 /etc/default/grub 并设置 GRUB_DEFAULT=0.意思是 GRUB 初始化页面的第一个内核将作为默认内核.
$ cat /etc/default/grub |grep GRUB_DEFAULT
GRUB_DEFAULT=0

# 重新生成内核配置
$ grub2-mkconfig -o /boot/grub2/grub.cfg

# 重启机器
$ reboot


```





2. `Nameserver limits were exceeded`

```bash
# kubelet 的nameserver数量超出限制,不能超过3个
$ cat /etc/resolv.conf
```



3. docker私有仓库，本地认证

```bash
#将Harbor 里面的CA证书,放在指定两个位置,并重启docker
$ sudo mkdir /etc/docker/certs.d/harbor.devops.nari
$ sudo cp ca.crt /etc/docker/certs.d/harbor.devops.nari/
$ sudo cat ca.crt >> /etc/ssl/certs/ca-certificates.crt
$ sudo systemctl restart docker
```



4.  kubelet报错

```bash
 kubelet_network_linux.go:111] Not using --random-fully in the MASQUERADE rule for iptables because the local version of iptables does not support it
```

解决

```bash
#安装所需的依赖
$ yum install gcc make libnftnl-devel libmnl-devel autoconf automake libtool bison flex  libnetfilter_conntrack-devel libnetfilter_queue-devel libpcap-devel
$ export LC_ALL=C
$ wget wget https://www.netfilter.org/projects/iptables/files/iptables-1.6.2.tar.bz2
$ tar -xvf iptables-1.6.2.tar.bz2
$ cd iptables-1.6.2 \
 ./autogen.sh \
 ./configure \
 make -j4 \
 make install
 #当然可以把cd /usr/local/sbin下面的iptables相关的东西打包然后分发到其它服务器
 $ cd /usr/local/sbin
 $ \cp iptables /sbin
 $ \cp iptables-restore /sbin/
 $ \cp iptables-save /sbin/
 
 # 重启kube-proxy
```



