
## 1.注意事项
k8s安装有多种方式，官方支持的为`kubeadm`，可以参考本文档中的[教程](kubeadm.md)快速安装。由于kubeadm可以快速安装一个k8s集群，但是在生产级别情况下，建议使用`kubespray`安装,可以参考本文教程[kubespray](kubespray.md)

> 若想获取更详细信息可参考
> - kubeadm: [https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm/](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
> - kubespray: [https://kubespray.io/#/](https://kubespray.io/#/)

如果想纯二进制安装可以参考[https://github.com/w-huaqiang/zdgtdui](https://github.com/w-huaqiang/zdgtdui)

一般有人会问kubeadm安装好还是二进制安装好，对于这个问题没有绝对答案，只要你对其熟悉即可。kubeadm是官方推荐的安装方式，即使你有自己的解决方案，依然可以基于kubeadm之上集成。