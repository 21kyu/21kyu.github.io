---
title: Kubernetes의 Watch event는 어떻게 동작하는가
author: wq
name: Wongyu Lee
link: https://github.com/kyu21
date: 2021-12-26 16:00:00 +0900
categories: [kubernetes, analyze]
tags: [kubernetes, watch, event]
render_with_liquid: false
---

대부분 이미 Kubernetes의 Watch 기능을 사용해봤을 것이다.
나는 Deployment 등을 배포하고 Pod의 배포 상태를 계속 보고 싶을 때 Kubectl에 *-w* 옵션을 추가해 확인하곤 했다.
또한 Controller를 구현할 때에도 Watch() Method를 사용하면 결과적으로 channel을 통해 event를 받아올 수 있어 변경 감지에 따른 동작을 정의만 해주면 됐다.

Kubernetes에서는 이러한 Watch 기능을 사용하면 API server로부터 데이터를 지속적으로 전달받는다는걸 알겠는데, 정확히 어떠한 방식으로 동작되는지 궁금해졌으므로 차근히 확인하면서 여기에 정리해놓고자 한다.
혹 글 내용에 대한 수정이 필요하다면 계속해서 업데이트할 예정이다.

*Last updated: 2021/12/26*

### Prerequisites

- [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
- [minikube](https://minikube.sigs.k8s.io/docs/) and *minikube start*

## Watch pods via kubectl

먼저 Watch 기능을 사용해보도록 하자.
익히 알고 있는 바와 같이 한 줄의 Kubectl command로 cluster 상에 있는 (default namespace의) Pod들을 감시할 수 있게 된다.
```shell
❯ kubectl get pods --watch
```
{: .nolineno}

새로운 shell을 열어 `nginx` pod 하나를 배포한다.

```shell
❯ kubectl run nginx --image=nginx
pod/nginx created
```
{: .nolineno}

이제 다시 *watch*를 걸어뒀던 shell로 돌아가면 `nginx` pod의 상태(status)가 변하는 과정을 관찰할 수 있다.
```shell
NAME    READY   STATUS              RESTARTS   AGE
nginx   0/1     Pending             0          0s
nginx   0/1     Pending             0          0s
nginx   0/1     ContainerCreating   0          0s
nginx   1/1     Running             0          14s
```
{: .nolineno}

어떻게 받아오는 것인지 궁금하기에 조금 더 깊게 들여다 봐야겠다. 로그 수준[^1]을 최대로 높여서 *watch* command를 걸어둔 상태로 `nginx-2` pod를 추가 배포하면 아래와 같은 로그를 만나게 된다.

```shell
❯ kubectl get pods --watch -v 9
...
GET https://127.0.0.1:57226/api/v1/namespaces/default/pods?limit=500
...
Response Body: {"kind":"Table","apiVersion":"meta.k8s.io/v1","metadata":{"resourceVersion":"818"}, ...
...
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          4m43s
...
GET https://127.0.0.1:57226/api/v1/namespaces/default/pods?resourceVersion=818&watch=true
...

# kubectl run nginx-2 --image=nginx
nginx-2   0/1     Pending             0          0s
nginx-2   0/1     Pending             0          0s
nginx-2   0/1     ContainerCreating   0          0s
nginx-2   1/1     Running             0          5s
```
{: .nolineno}

로그 중 주의 깊게 봐야될 것 같은 부분을 추려봤다. 결국 Kubectl의 *watch* command는 이렇게 동작하는걸로 보인다.
1. Kubectl은 Kuberntes API server로 (Default Namespace의) Pod의 List를 요청한다. (여기서는 max 500으로 설정됨)
2. Pod List에 대한 응답을 받고 다시 GET 요청을 한다. 이 때, 응답의 Boby에 있었던 resourceVersion과 함께 watch도 쿼리 파라미터로 같이 보낸다.

Kubectl이 보여주는 로그를 통해 HTTP GET 요청에 `?watch` 쿼리 파라미터를 추가해 보내게 되면
Kubernetes는 이를 *get* 동작이 아닌 *watch* 동작으로 받아들이게 된다는 걸 어림잡아 알게 됐다.

이는 Reflector가 구현해둔 [ListAndWatch](https://pkg.go.dev/k8s.io/client-go/tools/cache#Reflector.ListAndWatch) Method의 동작과 정확히 일치하는 흐름이다.
왜 2번의 GET 요청을 하도록 구현되었을까? 그리고 `resourceVersion`은 무엇일까?

## List and Watch

Kubernetes에서는 효율적인 변경 추적을 위해, 모든 object에 존재하는 `resourceVersion` field를 사용한다고 한다.
etcd와 같은 persistence layer에 영속화되는 순간에 대한 상태를 식별할 수 있는 일종의 지문으로,
상태가 변경될 때마다 `resourceVersion`도 같이 변경된다.

|  | Pod List | nginx Pod | nginx-2 Pod |
|-------|--------|---------|---------|
| kubectl run nginx ... | 1920 | 628 | - |
| kubectl run nginx-2 ... | 2137 | 628 | 1939 |
| kubectl edit nginx ... | 6861 | 6855 | 1939 |

*: 상태 변경에 따른 resourceVersion 변화*

Kubectl과 같은 클라이언트들은 object 또는 collection에 대한 초기 요청(*get* 또는 *list*)의 응답으로 받은 `resourceVersion`을 사용하여 감시 요청(*watch*)을 하게 되면,
이후에 발생되는 *Create*, *Update*, *Delete* event와 같은 후속 변경 사항을 구독할 수 있다.
이러한 내용들 때문에 2번의 요청이 요구되는 것이다.
resourceVersion을 얻기 위해 첫번째 GET 요청을 하며, 해당 요청에 포함된 resourceVersion과 함께 두번째 GET 요청을 하면서 리소스에 대한 정확한 구독 시점이 결정되는 것이다.

*watch* 요청 시 전달되는 `resourceVersion`에 따라 변경 감지의 시작점이 달라지게 되는데
[Semantics for watch](https://kubernetes.io/docs/reference/using-api/api-concepts/#semantics-for-watch) 에서 보다 자세한 내용을 확인할 수 있다.

| resourceVersion unset | resourceVersion="0" | resourceVersion="{value other than 0}" |
|-------|--------|---------|
| Get State and Start at Most Recent | Get State and Start at Any | Start at Exact |

이제는 *watch event*의 동작에 대해서 어느정도 알긴 하겠다. 여기에 추가로, 어떤 방식으로 Kubernetes가 event를 주고 클라이언트가 받을 수 있게 되는지 구현에 대한 정리가 되면 더 명확해질 것으로 보인다.

## Client-side Implementation

클라이언트의 구현을 따라가보자. Kubectl의 [watch](https://github.com/kubernetes/kubectl/blob/d7da6ad9f193e3afdc754527c0a5ac9390c053fb/pkg/cmd/get/get.go#L638) 구현에서,
*watch* 요청이 오면 최종적으로 `request.go`에서 생성하는 `watch.go`의 `watch.Interface`에서 channel을 꺼내 event가 도착할 때마다 출력한다.
커스텀한 Controller 또는 Webhook을 구현할 경우 리소스에 대한 변경 감지가 필요한 상황(*e.g., Pod 생성 요청을 하고 생성이 완료될 때까지 대기*)이면 clientSet의 client를 통해 원하는 리소스에 대해 Watch verb를 요청해 원하는 로직을 구현할 수도 있는데[^2],
이 때 얻게 되는 `watch.Interface` 역시 같은 방식으로 만들어지는 객체이다.

```go
watch, err := client.CoreV1().Pods(namespace).Watch(context.TODO(), options)
// ...

for event := range watch.ResultChan() {
	pod, ok := event.Object.(*corev1.Pod)
	// ...

	switch event.Type {
	case watch.Added, watch.Modified, watch.Deleted, watch.Bookmark:
  		// ...
	case watch.Error:
  		// ...
  }
}
```

간단하게는 이러한 방식으로 client 객체를 통해 변경 감지가 필요한 리소스의 `watch.Interface`를 받을 수 있다.
`watch.ResultChan()`으로 event를 전달받을 수 있으며 Type에 따라 필요한 로직을 작성하면 된다.

클라이언트 Watcher를 위한 코드는 내부적으로 이런 형태로 구현돼 있다.

```go
func (r *Request) Watch(ctx context.Context) (watch.Interface, error) {
	// ...
	var retryAfter *RetryAfter
	url := r.URL().String()
	for {
		req, err := r.newHTTPRequest(ctx)
		if err != nil {
			return nil, err
		}

		// ...

		resp, err := client.Do(req)
		// ...
		if err == nil && resp.StatusCode == http.StatusOK {
			return r.newStreamWatcher(resp)
		}

		// ...
	}
}
```
{: file="request.go" }

`watch.Interface`의 prototype들을 구현한 구현체 중 하나인 StreamWatcher를 반환하는 [Watch(context.Context)](https://github.com/kubernetes/client-go/blob/v0.23.1/rest/request.go#L671) Method다.
Golang을 공부하며 그동안 예제로 보아왔던 HTTP streaming 클라이언트측 코드 구현과 매우 흡사한걸 확인할 수 있다.
streaming을 위한 `http.request` 객체를 만들고 요청을 보낸 후 응답을 받아 데이터를 지속적으로 수신받을 Decoder를 생성한다.
StreamWatcher 내부에 Decoder가 존재하며 아래와 같이 구성된다.

```go
func NewStreamWatcher(d Decoder, r Reporter) *StreamWatcher {
	sw := &StreamWatcher{
		source:   d,
		reporter: r,
		result: make(chan Event),
		done: make(chan struct{}),
	}
	go sw.receive()
	return sw
}
```
{: file="streamwatcher.go" }

Goroutine으로 실행되는 `sw.receive()`에서 Decoder로부터 Type과 Object를 얻고 result channel로 전송한다.
이러한 result channel은 `ResultChan()` Method를 통해 가져올 수 있으므로, 변경 감지가 필요한 클라이언트는 이를 사용해 channel로부터 [Event](https://pkg.go.dev/k8s.io/apimachinery/pkg/watch#Event) 를 수신할 수 있게 되는 것이다.

## Server-side Implementation

그렇다면 서버측은 어떻게 구현되어 있을까? 코드를 샅샅히 뒤져보면 WatchServer 구조체가 존재하며 [ServeHTTP](https://github.com/kubernetes/apiserver/blob/v0.23.1/pkg/endpoints/handlers/watch.go#L163) Method로 *watch* 응답을 제공한다는 것을 확인할 수 있다.

```go
func (s *WatchServer) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	// ...

	if wsstream.IsWebSocketRequest(req) {
		w.Header().Set("Content-Type", s.MediaType)
		websocket.Handler(s.HandleWS).ServeHTTP(w, req)
		return
	}

	// ...

	e := streaming.NewEncoder(framer, s.Encoder)

	// ...

	w.Header().Set("Content-Type", s.MediaType)
	w.Header().Set("Transfer-Encoding", "chunked")
	w.WriteHeader(http.StatusOK)
	flusher.Flush()

	// ...
	ch := s.Watching.ResultChan()
	done := req.Context().Done()

	for {
		select {
		// ...
		case event, ok := <-ch:
			// ...

			if err := e.Encode(outEvent); err != nil {
				utilruntime.HandleError(fmt.Errorf("unable to encode watch object %T: %v (%#v)", outEvent, err, e))
				return
			}
			// ...
		}
	}
}
```
{: file="watch.go" }

우선 요청에서 Connection과 Upgrade 헤더를 봐서 WebSocket 연결 요청인지 확인한다.
맞다면 WebSocket connection을 통해, 아니라면 응답의 Transfer-Encoding 헤더를 *chunked*로 설정하여 streaming HTTP connection을 통해서 일련의 인코딩된 event를 클라이언트에게 제공한다.

코드를 보면 WatchServer 또한 특정 channel로부터 event를 수신해서 최종적으로 클라이언트에게 송신하는 중간 전달자임을 확인할 수 있다.

`ch := s.Watching.ResultChan()`

음.. Watching 객체도 `watch.Interface`로 WatchServer가 생성될 때 외부에서 주입받아 사용이 되는데 이건 또 어디에서 오는걸까?
이 부분은 다음에 Kubernetes의 `Event`에 대해 면밀히 정리를 해보면서 추가로 확인해보고자 한다.

## Watch event flow

개략적인 Watch event의 흐름은 **그림 1**과 같다.
![Watch event flow](/images/watch-event-flow.png)
_그림 1. Watch event flow_

client-go의 clientSet을 통해 `Watch()` Method를 호출해도 그림 상의 `Request.Watch`에 도달하는 것은 동일하다.

## Conclusion

Kubernetes에서의 Watch event는 어떤 방식으로 구현돼 있는지 어떻게 동작하는지 어느정도 정리해보았다.
예상했던 바와 같이 클라이언트와 API server는 Go-based HTTP streaming via HTTP & websocket으로 리소스에 대한 add/update/delete 타입의 event를 전달하고 받고 있었다.

예를 들어 Pod가 생성되거나 변경되거나 삭제가 되면 일련의 event가 발생되어 전달을 받고 해당 event에 대한 부가적인 처리를 할 수 있다는 아주 기본적이면서 당연한 말이긴 하다.
Kubernetes에서는 event를 내부적으로 어떤 방식으로 처리하게 했는지에 대해 알게 되면 추후에 도움이 되지 않을까 싶어서 다음 순으로는 Kubernetes의 `Event`에 대해서 정리해보고자 한다.

<div style="text-align: center; margin-top: 100px; margin-bottom: 50px">- 끝. -</div>

---
[^1]: kubectl은 -v 또는 --v 플래그를 통해 로그 수준을 지정할 수 있도록 지원하고 있다. [kubectl-output-verbosity-and-debugging](https://kubernetes.io/docs/reference/kubectl/cheatsheet/#kubectl-output-verbosity-and-debugging) 에서 자세히 확인할 수 있다.
[^2]: *Watch*보다는 *Informer* 사용이 권장된다.
