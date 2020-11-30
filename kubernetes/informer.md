# comment
- [comment](#comment)
  - [1. 控制循环](#1-控制循环)
  - [2. 自定义controller事例](#2-自定义controller事例)

## 1. 控制循环
在k8s中存在各种controller，其中包括自身的controller manager，和自定义controller，其中自定义的controller是用来扩展k8s。
可以说controller是k8s的灵魂，那么k8s怎么使用controller做到的控制循环，我们通过分析client-go中informer来理解，为了更好的理解，这里举例将replicaSet A的 replicas 从1 变成2。

```bash
api Server  <------------------------------------------------------------------|
    |                                                                          |
    ↓                                                                          |
Reflector--->Delta Queue--->informer--->indexer--->[local store cache]         |
                             |             ↑                                   |
                             |             |---------                          |
    Add handler <-------------                      |                          |
    update handler                                  |                          |
    delete handler                                worker                       |
        |—------------------> work queue--------> worker-----------------------|
                                    |             worker
                                    |                |
                                    <--- if faild ---|
```

- reflector 通过list/watch APIServer，发现replicaSet A有变化(update)，将replicaSet A信息 + update存入 Delta Queue。
- informer 从Delta Queue中取出replicaSet A，并将其更新后的信息交给indexer，indexer将其存入本地缓存 local store
- informer 将信息交给indexer之后，触发update handler Func。update handler Func将replicaSet A存入work queue队列
- work queue后面的一个work取出replicaSet A的name，并且从缓存中取出replicaSet A的信息（包含status=1和spec=2）,发现需要创建一个pod，于是就向apiServer发送请求，创建一个pod，并且把ownerRefence指向replicaSet A

- 这时reflector 通过list/watch APIServer发现pod A有变化(create),将pod A信息 + create存入 Delta queue
- informer从Delta Queue中取出pod A，并将其创建信息交给indexer，indexer将其存入本地缓存 local store
- informer 将信息交给indexer之后，触发create hander Func。create handler Func通过判断pod ownerReferences，发现为replicaSet A，于是就将replicaSet A存入work queue
- worker queue后面的一个work取出replicaSet A的name，并且从缓存中取出replicaSet A的信息（包含status=1和spec=2），发现当前status应该为2,于是在此时更新status使得 spec 和 status 达成一致。



## 2. 自定义controller事例
```go
package main

import (
	"flag"
	"fmt"
	"log"
	"path/filepath"

	V1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/informers"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/cache"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/util/homedir"
)

func main() {
	var kubeconfig *string
	if home := homedir.HomeDir(); home != "" {
		kubeconfig = flag.String("kubeconfig", filepath.Join(home, ".kube", "config"), "(optional) absolute path to the kubeconfig file")

	} else {
		kubeconfig = flag.String("kubeconfig", "", "absolute path to the kubeconfig file")
	}

	flag.Parse()

	config, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)
	if err != nil {
		panic(err.Error())
	}

	clientset, err := kubernetes.NewForConfig(config)

	if err != nil {
		log.Panic(err.Error())
	}

	factory := informers.NewSharedInformerFactory(clientset, 0)

	informer := factory.Core().V1().Pods().Informer()

	stopCh := make(chan struct{})
	defer close(stopCh)

	informer.AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    onAdd,
		UpdateFunc: onUpdate,
	})

	go informer.Run(stopCh)

	if !cache.WaitForCacheSync(stopCh, informer.HasSynced) {
		log.Panic(fmt.Errorf("time out"))

	}

	<-stopCh
}

func onAdd(obj interface{}) {
	pod := obj.(V1.Object)

	log.Printf("New pod Added to Store: %s", pod.GetName())
	if annotation := pod.GetAnnotations(); annotation["app"] == "bjzdgt.com" {
		log.Println("This pod is a good pod")
	}

	_, ok := pod.GetLabels()["test"]

	if ok {
		fmt.Println("It has the label")
	}
}

func onUpdate(oldObj interface{}, newObj interface{}) {

	pod := newObj.(V1.Object)
	if annotation := pod.GetAnnotations(); annotation["app"] == "bjzdgt.com" {
		log.Println("This pod is a good pod")
	}
	log.Println(pod.GetName())

}

```

```bash
thomaswangdeMacBook-Pro:k8s thomas$ go run main.go 
...
2020/11/30 17:30:31 New pod Added to Store: csi-cephfsplugin-provisioner-58c4f6c77f-8dr4w
2020/11/30 17:30:31 New pod Added to Store: nodelocaldns-tx4sp
2020/11/30 17:30:31 New pod Added to Store: nginx
2020/11/30 17:30:31 This pod is a good pod
2020/11/30 17:30:31 New pod Added to Store: rook-ceph-osd-7-66f5d6f86f-672tp
2020/11/30 17:30:31 New pod Added to Store: nginx-proxy-node4
2020/11/30 17:30:31 New pod Added to Store: rook-ceph-mgr-a-5dc7b47dd-rq6rc
2020/11/30 17:30:31 New pod Added to Store: rook-discover-tjlmb
```