
# comment
- [comment](#comment)
  - [1. 介绍](#1-介绍)
  - [2. Main 入口](#2-main-入口)
  - [2. nfsProvisioner](#2-nfsprovisioner)


## 1. 介绍
k8s内部包含一些in-tree的存储使用，比如nfs，但是对于nfs 使用storage class这样动态分配，需要提供dynamic provisioner。通过此provisioner可以根据pvc动态创建pv,并且配置pv的nfs path。此文分析nfs-provisioner源码来看如何开发一个自己的provisioner。

源码位于：https://github.com/kubernetes-retired/external-storage/blob/master/nfs-client/cmd/nfs-client-provisioner/provisioner.go

## 2. Main 入口
首先分析程序入口Main函数,看需要构建哪些对象
```go
func main() {

    // 解析命令行参数
    flag.Parse()
    
    //设置参数"logtostderr" 为true
	flag.Set("logtostderr", "true")

    //获取环境变量"NFS_SERVER"的值，如果为空则记录日志Fatal并退出
	server := os.Getenv("NFS_SERVER")
	if server == "" {
		glog.Fatal("NFS_SERVER not set")
    }
    
    //获取环境变量"NFS_PATH"的值，如果为空则记录日志Fatal并退出.
	path := os.Getenv("NFS_PATH")
	if path == "" {
		glog.Fatal("NFS_PATH not set")
    }
    
    //获取环境变量provisionerNameKey的值，如果为空则记录日志Fatal并退出.
    //provisionerNameKey为一个常量 = "PROVISIONER_NAME"
	provisionerName := os.Getenv(provisionerNameKey)
	if provisionerName == "" {
		glog.Fatalf("environment variable %s is not set! Please set it.", provisionerNameKey)
	}

	// Create an InClusterConfig and use it to create a client for the controller
    // to use to communicate with Kubernetes
    // 创建一个client用于和k8s交互，此处使用的是InClusterConfig，即默认此程序运行在k8s中，通过k8s的service account获取k8s的认证
    //rest 为导入的库 k8s.io/client-go/rest
	config, err := rest.InClusterConfig()
	if err != nil {
		glog.Fatalf("Failed to create config: %v", err)
    }
    
    //实例化一个client，如果有错误则记录日志
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		glog.Fatalf("Failed to create client: %v", err)
	}

	// The controller needs to know what the server version is because out-of-tree
    // provisioners aren't officially supported until 1.5
    //获取k8s版本,如果有错误则记录日志
	serverVersion, err := clientset.Discovery().ServerVersion()
	if err != nil {
		glog.Fatalf("Error getting server version: %v", err)
	}

    //定义一个nfsprovisioner对象，此对象为一个结构体
    // client: 一个实例化的clientSet
    // server: nfs server地址
    // path: nfs路径
	clientNFSProvisioner := &nfsProvisioner{
		client: clientset,
		server: server,
		path:   path,
	}
	// Start the provision controller which will dynamically provision efs NFS
    // PVs
    //定义一个provision controller
    // 其中provision相关对象通过 package "github.com/kubernetes-sigs/sig-storage-lib-external-provisioner/controller"已经定义好了
    //传入相关参数，包含一个clientSet，provision的名称,一个nfsprovision对象，k8s版本
    pc := controller.NewProvisionController(clientset, provisionerName, clientNFSProvisioner, serverVersion.GitVersion)
    
    //进入主循环运行，wait.NeverStop为package “"k8s.io/apimachinery/pkg/util/wait"中对象
	pc.Run(wait.NeverStop)
```

相关注视已经在代码中，这里面的核心为最后两行
```go
pc := controller.NewProvisionController(clientset, provisionerName, clientNFSProvisioner, serverVersion.GitVersion)
pc.Run(wait.NeverStop)
```
1. 通过自定义的provision构建provisionController，其中provision必须实现provisioner借口(即，定义的nfsProvisioner,必须包含Provision和Delete两个方法)
2. 执行ProvisionController的Run方法。


## 2. nfsProvisioner
通过上面main函数我们看出，需要定义一个对象nfsProvisioner，且nfsProvisioner需要实现Provisioner接口

```go
// 定义nfsProvisioner结构体
type nfsProvisioner struct {
	client kubernetes.Interface
	server string
	path   string
}

// 定义两个常量
//provisionerNameKey在main函数中被使用获取的一个环境变量的key,即必须在启动时设置key为"PROVISIONER_NAME"的环境变量
const (
	provisionerNameKey = "PROVISIONER_NAME"
)

const (
	mountPath = "/persistentvolumes"
)

// 在编译时验证是否nfsProvisioner实现了Provisioner接口
var _ controller.Provisioner = &nfsProvisioner{}

//定义Provision函数，其为Provisioner接口指定的方法，名字必须为Provision,且传入的参数和返回的参数必须如下格式
//传入 VolumeOptions参数，其中VolumeOptions结构体格式如下:
/*
type VolumeOptions struct {
    //PV对象的回收Policy,此处定义未用到
    PersistentVolumeReclaimPolicy v1.PersistentVolumeReclaimPolicy
    // PV.Name of the appropriate PersistentVolume. Used to generate cloud
    // volume name.
    // PV 名称
    PVName string

    // PV mount options. Not validated - mount of the PVs will simply fail if one is invalid.
    
    // PV 挂载选项，为一个切片
    MountOptions []string

    // PVC is reference to the claim that lead to provisioning of a new PV.
    // Provisioners *must* create a PV that would be matched by this PVC,
    // i.e. with required capacity, accessMode, labels matching PVC.Selector and
    // so on.
    // 对应PVC的对象，PVC相关信息均包含在此
    PVC *v1.PersistentVolumeClaim
    // Volume provisioning parameters from StorageClass

    //StorageClass传入的参数对象，为map格式,此处定义未用到
    Parameters map[string]string

    // Node selected by the scheduler for the volume.
    //被绑定的Node节点,此处定义未用到
    SelectedNode *v1.Node
    // Topology constraint parameter from StorageClass

    //相关的存储拓扑对象,此处定义未用到
    AllowedTopologies []v1.TopologySelectorTerm
}
*/
func (p *nfsProvisioner) Provision(options controller.VolumeOptions) (*v1.PersistentVolume, error) {

    // 如果PVC的Spec中Selector已经被指定则返回，说明未使用storageClass动态创建pv
	if options.PVC.Spec.Selector != nil {
		return nil, fmt.Errorf("claim Selector is not supported")
	}
	glog.V(4).Infof("nfs provisioner: VolumeOptions %v", options)

    //获取PVC的命名空间，名称
	pvcNamespace := options.PVC.Namespace
	pvcName := options.PVC.Name

    //通过PVC相关信息拼凑出PV的名称
	pvName := strings.Join([]string{pvcNamespace, pvcName, options.PVName}, "-")

    //创建nfs的路径，该路径给待创建的pv使用
    //此处说明，在启动nfs-provisioner的时候已经将nfs挂载到了该pod的mountPath路径下，对nfs的操作
    //均可直接命令创建目录路径
	fullPath := filepath.Join(mountPath, pvName)
	glog.V(4).Infof("creating path %s", fullPath)
	if err := os.MkdirAll(fullPath, 0777); err != nil {
		return nil, errors.New("unable to create directory to provision new pv: " + err.Error())
	}
	os.Chmod(fullPath, 0777)

    //连接nfsProvisioner提供的path和pvName，组合成完整路径
	path := filepath.Join(p.path, pvName)

    //定义完成的pv对象，类似pv的yaml文件
	pv := &v1.PersistentVolume{
		ObjectMeta: metav1.ObjectMeta{
			Name: options.PVName,
		},
		Spec: v1.PersistentVolumeSpec{
			PersistentVolumeReclaimPolicy: options.PersistentVolumeReclaimPolicy,
			AccessModes:                   options.PVC.Spec.AccessModes,
			MountOptions:                  options.MountOptions,
			Capacity: v1.ResourceList{
				v1.ResourceName(v1.ResourceStorage): options.PVC.Spec.Resources.Requests[v1.ResourceName(v1.ResourceStorage)],
			},
			PersistentVolumeSource: v1.PersistentVolumeSource{
				NFS: &v1.NFSVolumeSource{
					Server:   p.server,
					Path:     path,
					ReadOnly: false,
				},
			},
		},
	}
	return pv, nil
}

//定义Delete函数，其为Provisioner接口指定的方法，名字必须为Delete,且传入的参数和返回的参数必须如下格式
//传入 *v1.PersistentVolume参数为一个pvd对象:
func (p *nfsProvisioner) Delete(volume *v1.PersistentVolume) error {

    // 获取pv中指定的nfs path
    path := volume.Spec.PersistentVolumeSource.NFS.Path
    
    //获取该pv在nfs path中对应的名称：应该为pvcnamespace-pvcname-pvname
    //获取目录下的完整路径
	pvName := filepath.Base(path)
    oldPath := filepath.Join(mountPath, pvName)
    
    //判断相关目录是否存在
	if _, err := os.Stat(oldPath); os.IsNotExist(err) {
		glog.Warningf("path %s does not exist, deletion skipped", oldPath)
		return nil
	}
	// 通过传入的pv信息获取storageClass的信息.该函数后面有说明
	storageClass, err := p.getClassForVolume(volume)
	if err != nil {
		return err
	}
	// Determine if the "archiveOnDelete" parameter exists.
	// If it exists and has a false value, delete the directory.
    // Otherwise, archive it.
    //获取storageClass是否包含"archiveOnDelete"参数,若包含参数且为False, 则删除目录
	archiveOnDelete, exists := storageClass.Parameters["archiveOnDelete"]
	if exists {
		archiveBool, err := strconv.ParseBool(archiveOnDelete)
		if err != nil {
			return err
		}
		if !archiveBool {
			return os.RemoveAll(oldPath)
		}
	}

    // 若上面判断archiveOnDelete为true，则重命名目录，并归档相关的目录
	archivePath := filepath.Join(mountPath, "archived-"+pvName)
	glog.V(4).Infof("archiving path %s to %s", oldPath, archivePath)
	return os.Rename(oldPath, archivePath)

}

// getClassForVolume returns StorageClass
//辅助函数，其为通过pv信息获取对应storageClass相关信息
func (p *nfsProvisioner) getClassForVolume(pv *v1.PersistentVolume) (*storage.StorageClass, error) {
    // 判断nfsprovisioner是否包含初始化好的clientset
	if p.client == nil {
		return nil, fmt.Errorf("Cannot get kube client")
    }
    
    //获取storageClasssName
    //GetPersistentVolumeClass为package "k8s.io/kubernetes/pkg/apis/core/v1/helper"中函数
	className := helper.GetPersistentVolumeClass(pv)
	if className == "" {
		return nil, fmt.Errorf("Volume has no storage class")
    }
    //获取指定storageclass name对应的storageClass对象，并且返回
	class, err := p.client.StorageV1().StorageClasses().Get(className, metav1.GetOptions{})
	if err != nil {
		return nil, err
	}
	return class, nil
}

```