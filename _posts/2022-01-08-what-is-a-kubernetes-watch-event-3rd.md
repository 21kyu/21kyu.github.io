---
title: Informer의 구조
author: wq
name: Wongyu Lee
link: https://github.com/kyu21
date: 2022-01-08 22:45:00 +0900
categories: [kubernetes, analyze]
tags: [kubernetes, watch, event, apiserver, gorestful, informer]
render_with_liquid: false
---

일전에 정리했던 내용과 같이 클라이언트가 API server에 Watch event를 요청할 때 streaming HTTP connection을 맺어
해당 리소스의 변경에 대한 이벤트를 전달받곤 했다.
이러한 변경 감지가 필요할 때마다 API server에 접근해야하는건 Kubernetes 시스템에 부하를 줄 수도 있다.
게다가 여러 클라이언트, 컨트롤러에서 Watch event를 요청하게 되면 시스템에 더욱 높은 부하가 생성될 것이다.

Informer는 In-memory 캐싱을 통해 이러한 문제를 해결하고자 했다. 이번 포스팅에서는 Informer에 대해 정리해보도록 하자.

### Prerequisites

- [Go](https://go.dev)
- [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
- [minikube](https://minikube.sigs.k8s.io/docs/)

## First, let's use Informer

Informer에 대해 자세히 파악하기 전에 간단히 사용해보고자 한다.
먼저 minikube를 실행시켜주자.

```shell
❯ minikube start
😄  minikube v1.24.0 on Darwin 12.2
✨  Automatically selected the docker driver. Other choices: hyperkit, parallels, ssh
👍  Starting control plane node minikube in cluster minikube
🚜  Pulling base image ...
💾  Downloading Kubernetes v1.22.3 preload ...
    > preloaded-images-k8s-v13-v1...: 501.73 MiB / 501.73 MiB  100.00% 55.37 Mi
    > gcr.io/k8s-minikube/kicbase: 355.78 MiB / 355.78 MiB  100.00% 14.47 MiB p
🔥  Creating docker container (CPUs=2, Memory=1985MB) ...
🐳  Preparing Kubernetes v1.22.3 on Docker 20.10.8 ...
    ▪ Generating certificates and keys ...
    ▪ Booting up control plane ...
    ▪ Configuring RBAC rules ...
🔎  Verifying Kubernetes components...
    ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
🌟  Enabled addons: storage-provisioner, default-storageclass
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```
{: .nolineno}

Informer 예제를 작성하기 위한 Go 프로젝트를 하나 만들어준다.

```shell
❯ go mod init example.com/informer
go: creating new go.mod: module example.com/informer

❯ touch main.go
```
{: .nolineno}

**main.go** 파일 생성 후 아래 코드를 작성하도록 하자.

```go
package main

import (
	"fmt"
	"os"
	"time"

	v1 "k8s.io/api/core/v1"
	"k8s.io/client-go/informers"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/cache"
	"k8s.io/client-go/tools/clientcmd"
)

func main() {
	kubeconfigPath := "location to your kubeconfig file path"
	config, err := clientcmd.BuildConfigFromFlags("", kubeconfigPath)
	if err != nil {
		fmt.Printf("The kubeconfig can't be loaded: %v\n", err.Error())
		os.Exit(1)
	}

	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		fmt.Printf("NewForConfig failed: %v\n", err.Error())
		os.Exit(1)
	}

	stopCh := make(chan struct{})
	defer close(stopCh)

	informerFactory := informers.NewSharedInformerFactory(clientset, time.Second*1)

	podInformer := informerFactory.Core().V1().Pods()
	podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			pod := obj.(*v1.Pod)
			fmt.Println("Add: " + pod.Name)
		},
		UpdateFunc: func(oldObj, newObj interface{}) {
			pod := newObj.(*v1.Pod)
			fmt.Println("Update: " + pod.Name)
		},
		DeleteFunc: func(obj interface{}) {
			pod := obj.(*v1.Pod)
			fmt.Println("Delete: " + pod.Name)
		},
	})

	informerFactory.Start(stopCh)
	fmt.Println("InformerFactory started")

	<-stopCh
}
```

InformerFactory로부터 PodInformer를 가져와서 변경 감지에 대한 처리를 할 수 있는 기본적이고 간단한 코드라고 할 수 있겠다.
`main()` 함수를 하나하나 살펴보도록 하자.

16-20 라인은 `*rest.Config`를 가져오는 부분이다.
Kubernetes API server에 대한 접근하기 위한 여러 정보를 포함하고 있다.
각자의 로컬에 있는 **.kube/config** 파일을 통해 빌드를 하는데 한가지 참고할 부분은 `kubeconfigPath`로
**~/.kube/config** 등의 상대 경로가 아닌 절대 경로로 입력해 주어야 할 수도 있다.

23-27 라인에서는 `*rest.Config`를 통해 `*kubernetes.Clientset`을 생성하고 있다.
모든 기본 Kubernetes 리소스에 대한 많은 클라이언트가 여기에 포함되어 있기 때문에 Clientset라고 한다.

32 라인에서 [SharedInformerFactory](https://pkg.go.dev/k8s.io/client-go/informers@v0.23.1#SharedInformerFactory) 를 만든다.
Shared informer factory는 informer들이 애플리케이션 내에서 동일한 리소스의 변경 사항을 공유할 수 있게 한다.
즉, 서로 다른 제어 루프에서 Shared informer factory를 통해 API server에 대한 동일한 watch 커넥션을 사용할 수 있다는 말이다.
이러한 특성으로 인해서 kube-controller-manager 내부에 있는 수많은 컨트롤러가 리소스에 대해 접근을 각자 하지만
프로세스 내에 존재하는 단일 informer를 통해 리소스 변경에 대한 이벤트를 전달받게 된다.

이 후 Pod informer를 가져와서 `cache.ResourceEventHandlerFuncs`를 사용해
[ResourceEventHandler](https://pkg.go.dev/k8s.io/client-go/tools/cache@v0.23.1#ResourceEventHandler) 를 달아준다.
ResourceEventHandler를 통해 해당 리소스에서 발생되는 이벤트에 대한 알림을 처리할 수 있다.
한 가지 명심해야할 사안은 핸들러 내부에서 수신된 객체를 직접 수정해서는 안된다.

OnAdd
: Object가 추가될 때 호출된다.

OnUpdate
: Object가 수정될 때 호출된다. oldObj는 object의 변경에 따른 이벤트를 받기 전에 마지막으로 관찰된 상태이다.
여러 변경 사항이 결합되어 이벤트가 호출될 수 있으므로 이것을 단일 변경 사항에 대한 이벤트라 볼 순 없다.

OnDelete
: Object가 삭제되기 전의 상태를 가져올 수 있으면 가져오고 그렇지 못한다면 DeletedFinalStateUnknown 유형의 object를 가져오게 된다.
후자의 경우는 watch event가 어떠한 문제로 인해 종료되어 삭제에 대한 이벤트를 놓쳤을 경우에 발생될 수 있다.

위 이벤트에 대한 처리 로직을 작성하면 된다.
여기에선 간단히 로깅만 하는 정도로 해두었다.

마지막으로 53 라인에서 stop channel이 닫힐 때까지는 프로세스가 유지될 수 있도록 채널 수신을 받게 해두었다.
이제 바로 실행해보자.

```shell
❯ go mod tidy

❯ go run main.go
InformerFactory started
Add: kube-apiserver-minikube
Add: storage-provisioner
Add: coredns-78fcd69978-424p5
Add: kube-proxy-crjsd
Add: etcd-minikube
Add: kube-scheduler-minikube
Add: kube-controller-manager-minikube
Update: kube-apiserver-minikube
Update: storage-provisioner
Update: coredns-78fcd69978-424p5
Update: kube-proxy-crjsd
Update: etcd-minikube
...
```
{: .nolineno}

Kubernetes 클러스터 위에 올라가 있는 모든 Pod들의 이벤트가 핸들링되고 있음을 확인할 수 있다.
모든 namespace의 pod를 관찰할 필요는 없으므로 관찰 범위를 줄여보자.

```go
namespaceOption := informers.WithNamespace("default")
informerFactory := informers.NewSharedInformerFactoryWithOptions(clientset, time.Second*1, namespaceOption)
```

32 라인의 informerFactory를 생성하는 부분을 위와 같이 변경하고 다시 실행하자.

```shell
❯ go run main.go
InformerFactory started

# 한층 조용해졌다. 새로운 shell을 열어 nginx pod를 default namespace에 배포해보자.
❯ kubectl run nginx --image=nginx
pod/nginx created

# 이제 다시 이전 shell로 돌아가보면
Add: nginx
Update: nginx
Update: nginx
Update: nginx
Update: nginx
Update: nginx
...
```
{: .nolineno}

nginx pod에 대한 이벤트가 핸들링되고 있음을 확인할 수 있다.
여기서 이상한 부분이 있다. OnUpdate 이벤트는 왜 계속 불려지고 있을까?

```go
func(obj interface{}) error {
  // from oldest to newest
  for _, d := range obj.(Deltas) {
    obj := d.Object
    if transformer != nil {
      var err error
      obj, err = transformer(obj)
      if err != nil {
        return err
      }
    }

    switch d.Type {
    case Sync, Replaced, Added, Updated:
      if old, exists, err := clientState.Get(obj); err == nil && exists {
        if err := clientState.Update(obj); err != nil {
          return err
        }
        h.OnUpdate(old, obj)
      } else {
        if err := clientState.Add(obj); err != nil {
          return err
        }
        h.OnAdd(obj)
      }
    case Deleted:
      if err := clientState.Delete(obj); err != nil {
        return err
      }
      h.OnDelete(obj)
    }
  }
  return nil
}
```

Informer에서는 [DeltaFIFO](https://pkg.go.dev/k8s.io/client-go/tools/cache#DeltaFIFO) 에서 delta를 꺼내와 위와 같은 처리를 하게 된다.
13 라인부터의 switch 문을 보게 되면 Updated 타입 뿐만 아니라 Sync, Replaced와 같은 다른 타입 또한 `h.OnUpdate()`로 호출될 수 있는 것을 볼 수 있다.

Sync 타입은 informerFactory를 생성할 때 `defaultResync` 인자로 넘겨준 time.Duration과 밀접한 연관이 있다.
주기적으로 재동기화가 되며 이 기간동안 합성된 이벤트가 핸들러로 전달되게 된다.
위 작성된 코드에 따르면 1초마다 resync를 한다. 5초로 변경해보고 얼마나 자주 불리는지 확인해보자.

Replaced 타입은 watch event의 오류로 인해 re-list 작업을 해야할 때 발생된다. 앞서 정리했던 ListAndWatch 작업에서의 List라고 보면 된다.

이와 같은 다양한 변경 타입으로 인해 OnUpdate 이벤트는 지속적으로 호출되고 있는 것이다.
진짜 변경이 일어났을 경우에만 처리를 하고 싶다면 어떻게 해야할까?
우린 이미 `resourceVersion`이 무엇을 의미하는지 앞선 정리를 통해 알고 있는 상태이다.
당장 사용해보자.

```go
UpdateFunc: func(oldObj, newObj interface{}) {
  old := oldObj.(*v1.Pod)
  cur := newObj.(*v1.Pod)
  fmt.Println("Update: " + cur.Name + ", old RV: " + old.ResourceVersion + ", new RV: " + cur.ResourceVersion)
},
```

UpdateFunc을 이렇게 변경하고 다시 실행해보자.

```shell
❯ go run main.go
InformerFactory started
Add: nginx
Update: nginx, old RV: 2206, new RV: 2206
Update: nginx, old RV: 2206, new RV: 2206
Update: nginx, old RV: 2206, new RV: 2206
...
```
{: .nolineno}

old/new resourceVersion을 확인할 수 있다.
object에 대한 변경이 전혀 없는데도 OnUpdate의 동작으로 인해 꾸준히 핸들링되고 있다.

이제 nginx pod에 변경을 가해보자.
간단히 label을 추가하는 작업을 해보고자 한다.

```shell
❯ kubectl label pods nginx foo=bar
pod/nginx labeled
---
Update: nginx, old RV: 2206, new RV: 2451 <- pod/nginx labeled
Update: nginx, old RV: 2451, new RV: 2451
Update: nginx, old RV: 2451, new RV: 2451
```
{: .nolineno}

nginx pod에 label이 추가되고 resourceVersion 또한 변경되었다.
이렇듯 object가 영속화되는 시점의 상태를 식별할 수 있는 resourceVersion을 사용하면 쉽게 변경 감지에 대한 이벤트를 처리할 수 있다.

## Informer's Architecture

이제 Informer가 어떤 형태로 구성되어 있는지 살펴보도록 하자.
Kubernetes 소스 코드에 따른 Informer를 통한 watch event 전달 방식은 그림 1.과 같다.

![informer-architecture](/images/informer-architecture.png)
_그림 1. Informer가 watch event를 전달하는 방식_

하나의 SharedInformer는 특정 API Group 및 kind/resource의 object에 대한 연결을 제공한다.
object는 API Group, kind/resource, namespace 및 name으로 식별된다.

SharedInformer를 기반으로 Indexer 추가 및 가져오기 기능을 제공하는 SharedIndexInformer를 살펴보자.
(informers.NewSharedInformer()를 호출하면 NewSharedIndexInformer를 반환한다.)
SharedIndexInformer는 주요 컴포넌트인 Indexer, Controller, SharedProcessor를 확인하고 넘어가는 것이 좋겠다.

Indexer
: Indexer는 인덱싱된 로컬 캐시이다.
object를 저장하고 처리하는 인터페이스인 Store를 여러 인덱스로 확장하며 DeltaFIFO에게는 KnownObjects 역할을 하며
object를 조회할 때 사용되는 Key의 목록을 제공해준다.
또한 클라이언트가 Lister를 통해 List/Get 요청을 하면 캐싱된 데이터를 전달해준다.

Controller
: Controller는 실행될 때 지정된 리소스를 감시하고 모든 변경 사항이 지정된 Store에 저장되도록 하는 Reflector를 생성하면서 감시 작업에 ListerWatcher를 사용하도록 한다.
ListerWatcher는 object/notification들을 가져오고 이를 DeltaFIFO로 push하는 동시에 해당 FIFO에서 pop하여
HandleDeltas로 Delta를 처리하는 컨트롤러이며 각 Delta에 대해 Indexer를 업데이트하고 관련 알림을 sharedProcessor에 채운다.

SharedProcessor
: Informer의 모든 클라이언트(Listener를 통해)에 해당 알림을 전달하는 역할을 한다.
클라이언트에서 ResourceEventHandler를 추가하면 SharedProcessor의 Listener에 추가가 되어 알림을 지속적으로 받을 수 있게 된다.

여기에서 DeltaFIFO는 주어진 객체의 변경 사항들에 대한 Delta를 항목으로 다루는 Queue이다.
Informer가 실행될 때 생성되며 Pop(), Get(), GetByKey(), List()와 같은 method들을 제공한다.

## Conclusion

Inforemr는 특정 리소스에 대한 변경 감지가 필요한 수많은 클라이언트가 존재하더라도 하나의 watch connection으로 처리될 수 있도록 설계되었다.
변경 감지가 필요한 클라이언트는 단지 ResourceEventHandler를 추가하기만 하면 된다.
또한 Indexer에 캐싱된 데이터를 Lister를 통해서 접근해 사용할 수 있다.

그리고 Informer는 watch connection이 끊어지면 WatchErrorHandler를 호출한 후 Backoff 한다.
만약 중단된 기간이 길어져서 etcd가 event를 데이터베이스에서 삭제를 해 event가 손실되는 경우, Informer는 모든 object를 다시 나열(re-list)한다.
클라이언트가 Informer를 사용하지 않고 Watch() verb를 사용했다면 이러한 작업들을 직접해야만 하거나, 그렇지 않다면 event를 놓치게 될 것이다.

이러한 이유들로 Informer의 사용이 권장되고 있다. 이제 마지막으로 Kubernetes의 Event에 대해 파악해보자.

<div style="text-align: center; font-weight: bold; margin-top: 100px; margin-bottom: 50px">끝.</div>
