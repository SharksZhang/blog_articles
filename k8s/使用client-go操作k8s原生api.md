#使用client-go操作k8s原生api并实现controller
[toc]
在工作中使用client-go实现了根据k8s集群中的pod的创建和删除事件完成pod逻辑端口的创建和产出，实现了一个createPortController。将期间总结到的知识分享给大家。
##1. client-go简介
在我们平时开发中，跟APIserver通讯的最常用的方式是使用kubectl和调用k8s的REST API。但这两种方式在可编程性和效率方面都不太令人满意。其实最好的方式，是模仿k8s的controller去实现自己的功能。
所以k8s有了client-go这个项目。k8s把写controller所需的clients,utilities等都放到了client-go这个repository里面。如果要实现controller，可以在里面找到所需要的工具。

官方对于项目结构，如下所示。我们开发主要使用到kubernetes包和tools/cache包。

    -     The kubernetes package contains the clientset to access Kubernetes API.
    -     The discovery package is used to discover APIs supported by a Kubernetes API server.
    -     The dynamic package contains a dynamic client that can perform generic operations on arbitrary Kubernetes API objects.
    -     The transport package is used to set up auth and start a connection.
    -     The tools/cache package is useful for writing controllers.


## 2.controller工作原理
kube-controller大致结构如下图所示，典型的controller有一个或者多个informer,来跟踪某一个resource,跟APIserver保持通讯，把最新的状态反映到本地的cache中。只要这些资源有变化，informer会调用callback。这些callback只是做一些非常简单的预处理，把不关心的变化过滤，然后然后把关心的变更的object放到workqueue里面。其实，真正的业务逻辑都是在worker里面，一般一个Controller会启动很多goroutins跑Workers,处理workqueue里面的items。它会计算用户想要达到的状态和当前状太有多大区别。然后通过clients向APIserver发送请求。图里面红色部分是用户需要实现的，蓝色是client-go原件。
![](k8sController.JPEG)

## 3.操作原生api
我们要操作k8s的原生api，主要要使用到clientset，clientSet位于kubernetes包中，封装了kubernetes的所有api。
####3.1clientSet初始化
client初始化需要需要使用kubeconfig的绝对路径，一般路径为$HOME/.kube/config。该文件主要用来配置本地连接的kubernetes。
然后通过获取到rest.config对象，可以创建出clientset对象。
```bash
	kubeconfig := flag.String("kubeconfig", "", "absolute path to the kubeconfig file")
	flag.Parse()
	if *kubeconfig == "" {
		panic("-kubeconfig not specified")
	}
	config, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)
	if err != nil {
		panic(err)
	}
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err)
	}
```
#####3.1.1clientSet操作
clientset的用法比较简单。使用时，先选group，比如core，再选具体的resource，比如pod后者job，最后再把动词(create,get)填上。
```bash
	clientset.CoreV1().Pods("default").Get("example-xxxxx", metav1.GetOptions{})
```
下面在讲一下各个操作细微的差别。
```
	Get(name string, options meta_v1.GetOptions) (*v1.Pod, error)

```
在get的GetOptions中，有一个属性是ResourceVersion。如果没有设置resource version，api-server收到请求后会从etcd读取最新的值。但设为0的话，apiserver就会从local的cache里面把值读取出来。cache的值会有一定延迟。

```
	List(opts meta_v1.ListOptions) (*v1.PodList, error)

```
list也提供了ListOptions，功能和get的基本相似。
```
	Watch(opts meta_v1.ListOptions) (watch.Interface, error)

```
watch里面也有一个listoption，里面的resourceversion意义不一样，在watch的时候，apiserver会从这个resourceversion开始所有变化。这个一般永远要设置resourceversion，如果不设置，APIserver会从cache里随便一个时间点开始推送。
在informer里，一般都是先list，把resourceversion设为0,API Server就可以从cache里面list。List完成之后，把list的resource version取出来，并且设置成watch的listOption,这样就可以保证informer拿到的events连续。

## 4.实现简单controller
client-go在tools/cache里面提供了实现controller的工具。主要用到的是informer和workqueue。
####4.1informer
informer的输入就两个，一个是list function和watch function。而是informer所需要的callback。informer跑起来后，它会维护localstore。之后就可以直接访问localstore，而不用跟APIserver通讯。
informer还有一个好处，就是可靠性。如果有网络中断，informer会从断点开始继续watch，不会错过任何event。
在使用infomer写controller时，最好让informer都sync了再启动controller。

####4.2Workqueue
workqueue主要有两个作用：
1. 如果是同一个object，多次加到workqueue中，在dequeue时，只会出现一次。
2. 如果处理中出错，再加入workqueue中，不会被立即处理。

####4.3简单示例

首先定义一个需要创建的controller

``` java
type CreatePortsController struct {
	clientSet       *kubernetes.Clientset
	podController   cache.Controller
	podStoreIndexer cache.Indexer
	PodEventMap     *PodAndEventMap
}

```
通过cache.NewIndexerInformer创建出 informer并且传入回调函数cpc.enqueueCreatePod和cpc.enqueueDeletePod。当add事件时调用cpc.enqueueCreatePod，在delete事件时调用cpc.enqueueDeletePod，业务逻辑的入口都在这两个函数中。由于client-go原生的workqueue不符合我的需求，所有没有使用。自己定义了一个workqueue结构（PodAndEventMap）。
```java
func NewCreatePortsController() (*CreatePortsController, error) {
	cpc := &CreatePortsController{}

	cpc.PodEventMap = &PodAndEventMap{
		Event: make(map[string]*operatingPod),
	}
	cpc.clientSet = infra.GetClientset()
	if cpc.clientSet == nil {
		klog.Errorf("newCreatePortForPodController: monitorcommon.GetClientSet() kubernetes clientSet is nil")
		return nil, errors.New("kubernetes clientSet is nil")
	}
	watchlist := cache.NewListWatchFromClient(cpc.clientSet.CoreV1().RESTClient(), "pods", v1.NamespaceAll,
		fields.Everything())

	cpc.podStoreIndexer, cpc.podController = cache.NewIndexerInformer(
		watchlist,
		&v1.Pod{},
		time.Second*0,
		cache.ResourceEventHandlerFuncs{
			AddFunc:    cpc.enqueueCreatePod,
			DeleteFunc: cpc.enqueueDeletePod,
		},
		cache.Indexers{},
	)
	klog.Infof("create CreatePortForPodControlle successful")
	return cpc, nil
}

```

然后将controller run起来，此时这个controller可以监控k8s集群，并根据集群状态完成需要的任务。

```java

func (cpc *CreatePortsController) Run(workers int, stopCh <-chan struct{}) {
	defer runtime.HandleCrash()
	klog.Infof("CreatePortForPodController.Run : Starting serviceLookupController Manager ")
	go cpc.podController.Run(stopCh)
	var i int
	for i = 1; i < constvalue.WaitForCacheSyncTimes; i++ {
		if cache.WaitForCacheSync(stopCh, cpc.podController.HasSynced) {
			klog.Infof("cache.WaitForCacheSync(stopCh, cpc.podController.HasSynced) error")
			break
		}
		time.Sleep(time.Second * 1)
	}
	if i == constvalue.WaitForCacheSyncTimes {
		klog.Errorf("CreatePortForPodController.Run: cache.WaitForCacheSync(stopCh, cpc.podController.HasSynced:[%v]) error,", cpc.podController.HasSynced())
		return
	}

	klog.Infof("CreatePortForPodController.Run : Started podWorker")

	<-stopCh
	klog.Infof("Shutting down Service Lookup Controller")

}
```



