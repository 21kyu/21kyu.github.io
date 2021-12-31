---
title: Watch Server에 요청이 도달하기까지의 과정
author: wq
name: Wongyu Lee
link: https://github.com/kyu21
date: 2022-01-01 00:00:00 +0900
categories: [kubernetes, analyze]
tags: [kubernetes, watch, event, handler]
render_with_liquid: false
---

앞서 클라이언트와 API server가 어떤 방식으로 Watch event를 요청하고 응답하는지 확인해봤다.
이번에는 좀 더 깊게 들어가서 API server가 Watch event에 대한 응답을 하기 위해 어떤 준비를 하는지, Watch Server 생성까지의 구현 부분을 알아볼 차례이다.

분석하는 와중에 정말 많은 용어들이 등장했는데 내부적으로 왜 이러한 네이밍을 했는지 Ubiquitous Language를 선정할 때 오고 갔던 많은 내용들이 궁금해지기도 했다.
이러한 용어들도 같이 정리해볼까 한다.

### What is a Kubernetes Watch Event?

- [ ] 1. **Watch Event**: [Kubernetes의 Watch event는 어떻게 동작하는가](http://blog.wqlee.com/posts/what-is-a-kubernetes-watch-event/)
- [x] 2. **Resource Handler**: Watch Server에 요청이 도달하기까지의 과정
- [ ] 3. Informer
- [ ] 4. Event

### Prerequisites

- Kubernetes 소스 코드

## API Server analysis

[server.go](https://github.com/kubernetes/kubernetes/blob/7c013c3f64db33cf19f38bb2fc8d9182e42b0b7b/cmd/kube-apiserver/app/server.go#L84) 를 보면
여느 kubernetes component들과 마찬가지로 kube-apiserver는 cobra.Command [^1] 를 사용해 시작되는걸로 확인이 된다.

```go
func NewAPIServerCommand() *cobra.Command {
	s := options.NewServerRunOptions()
	cmd := &cobra.Command{
		Use: "kube-apiserver",
		// ...
		RunE: func(cmd *cobra.Command, args []string) error {
			// ...

			// set default options
			completedOptions, err := Complete(s)
			if err != nil {
				return err
			}

			// validate options
			if errs := completedOptions.Validate(); len(errs) != 0 {
				return utilerrors.NewAggregate(errs)
			}

			return Run(completedOptions, genericapiserver.SetupSignalHandler())
		},

  // ...

	return cmd
}
```

### ServerChain

cobra.Command의 RunE function에서 completedOptions를 설정하고 해당 옵션이 유효한지 확인한다.
그리고 지정된 APIServer를 생성하고 실행하는 `Run()` method를 호출한다.
`Run()` method 내부에서 호출되는 `CreateServerChain()` method를 살펴보자.

```go
func CreateServerChain(completedOptions completedServerRunOptions, stopCh <-chan struct{}) (*aggregatorapiserver.APIAggregator, error) {
	kubeAPIServerConfig, serviceResolver, pluginInitializer, err := CreateKubeAPIServerConfig(completedOptions)
	// ...
	apiExtensionsConfig, err := createAPIExtensionsConfig(*kubeAPIServerConfig.GenericConfig, kubeAPIServerConfig.ExtraConfig.VersionedInformers, pluginInitializer, completedOptions.ServerRunOptions, completedOptions.MasterCount,
		serviceResolver, webhook.NewDefaultAuthenticationInfoResolverWrapper(kubeAPIServerConfig.ExtraConfig.ProxyTransport, kubeAPIServerConfig.GenericConfig.EgressSelector, kubeAPIServerConfig.GenericConfig.LoopbackClientConfig, kubeAPIServerConfig.GenericConfig.TracerProvider))
	// ...

	apiExtensionsServer, err := createAPIExtensionsServer(apiExtensionsConfig, genericapiserver.NewEmptyDelegateWithCustomHandler(notFoundHandler))
	// ...
	kubeAPIServer, err := CreateKubeAPIServer(kubeAPIServerConfig, apiExtensionsServer.GenericAPIServer)
	// ...
	aggregatorConfig, err := createAggregatorConfig(*kubeAPIServerConfig.GenericConfig, completedOptions.ServerRunOptions, kubeAPIServerConfig.ExtraConfig.VersionedInformers, serviceResolver, kubeAPIServerConfig.ExtraConfig.ProxyTransport, pluginInitializer)
	// ...
	aggregatorServer, err := createAggregatorServer(aggregatorConfig, kubeAPIServer.GenericAPIServer, apiExtensionsServer.Informers)
	// ...
	return aggregatorServer, nil
}
```

completedOptions의 설정에 따라 kubeAPIServerConfig와 Extension API server를 위한 apiExtensionsConfig를 통해 기본 API server와 Extended API server를 만들고
생성된 두 API server의 access를 통합하는 Aggregator Server를 생성하여 반환한다.

*Aggregation Architecture에 대한 지식이 있으면 매우 유용할 것이라는 이야기를 전해들었기에 AggregatorServer와 관련해서
Aggregation layer [^2] 와 어떠한 연관이 있는지 추후에 확인해보도록 하자.*

이제 CreateKubeAPIServer method를 확인해보자.

```go
func CreateKubeAPIServer(kubeAPIServerConfig *controlplane.Config, delegateAPIServer genericapiserver.DelegationTarget) (*controlplane.Instance, error) {
	kubeAPIServer, err := kubeAPIServerConfig.Complete().New(delegateAPIServer)
	if err != nil {
		return nil, err
	}

	return kubeAPIServer, nil
}
```

*Line:2*의 `kubeAPIServerConfig.Complete().New(delegateAPIServer)` 코드가 보인다.
`Complete()` method를 통해 유효한 데이터가 존재해야하는 모든 설정되지 않은 필드를 채우고 `New()` method를 통해 kube-apiserver의 구성을 생성한다.

### APIServerHandlers

```go
func (c completedConfig) New(delegationTarget genericapiserver.DelegationTarget) (*Instance, error) {
	// ...

	s, err := c.GenericConfig.New("kube-apiserver", delegationTarget)
	if err != nil {
		return nil, err
	}

	if c.ExtraConfig.EnableLogsSupport {
		routes.Logs{}.Install(s.Handler.GoRestfulContainer)
	}

	// ...

	m := &Instance{
		GenericAPIServer:          s,
		ClusterAuthenticationInfo: c.ExtraConfig.ClusterAuthenticationInfo,
	}

	if c.ExtraConfig.APIResourceConfigSource.VersionEnabled(apiv1.SchemeGroupVersion) {
		legacyRESTStorageProvider := corerest.LegacyRESTStorageProvider{
			StorageFactory:              c.ExtraConfig.StorageFactory,
			ProxyTransport:              c.ExtraConfig.ProxyTransport,
			KubeletClientConfig:         c.ExtraConfig.KubeletClientConfig,
			EventTTL:                    c.ExtraConfig.EventTTL,
			ServiceIPRange:              c.ExtraConfig.ServiceIPRange,
			SecondaryServiceIPRange:     c.ExtraConfig.SecondaryServiceIPRange,
			ServiceNodePortRange:        c.ExtraConfig.ServiceNodePortRange,
			LoopbackClientConfig:        c.GenericConfig.LoopbackClientConfig,
			ServiceAccountIssuer:        c.ExtraConfig.ServiceAccountIssuer,
			ExtendExpiration:            c.ExtraConfig.ExtendExpiration,
			ServiceAccountMaxExpiration: c.ExtraConfig.ServiceAccountMaxExpiration,
			APIAudiences:                c.GenericConfig.Authentication.APIAudiences,
		}
		if err := m.InstallLegacyAPI(&c, c.GenericConfig.RESTOptionsGetter, legacyRESTStorageProvider); err != nil {
			return nil, err
		}
	}

	restStorageProviders := []RESTStorageProvider{
		apiserverinternalrest.StorageProvider{},
		authenticationrest.RESTStorageProvider{Authenticator: c.GenericConfig.Authentication.Authenticator, APIAudiences: c.GenericConfig.Authentication.APIAudiences},
		authorizationrest.RESTStorageProvider{Authorizer: c.GenericConfig.Authorization.Authorizer, RuleResolver: c.GenericConfig.RuleResolver},
		autoscalingrest.RESTStorageProvider{},
		batchrest.RESTStorageProvider{},
		certificatesrest.RESTStorageProvider{},
		coordinationrest.RESTStorageProvider{},
		discoveryrest.StorageProvider{},
		networkingrest.RESTStorageProvider{},
		noderest.RESTStorageProvider{},
		policyrest.RESTStorageProvider{},
		rbacrest.RESTStorageProvider{Authorizer: c.GenericConfig.Authorization.Authorizer},
		schedulingrest.RESTStorageProvider{},
		storagerest.RESTStorageProvider{},
		flowcontrolrest.RESTStorageProvider{},
		appsrest.StorageProvider{},
		admissionregistrationrest.RESTStorageProvider{},
		eventsrest.RESTStorageProvider{TTL: c.ExtraConfig.EventTTL},
	}
	if err := m.InstallAPIs(c.ExtraConfig.APIResourceConfigSource, c.GenericConfig.RESTOptionsGetter, restStorageProviders...); err != nil {
		return nil, err
	}

	// ...

	return m, nil
}
```

이 method는 먼저 *Line:4*의 `New("kube-apiserver", delegationTarget)`에서 APIServerHandlers를 생성한다.
여기에는 FullHandlerChain과 Director, 그리고 GoRestfulContainer와 NonGoRestfulMux가 포함된다.

GoRestfulContainer
: 클라이언트에서 pods, deployments, services 등의 리소스들에 접근할 때 사용되는 /apis/*와 같은 API들이 등록되며
[go-restful](https://pkg.go.dev/github.com/emicklei/go-restful#section-readme) design pattern을 준수하는
[*restful.Container](https://pkg.go.dev/github.com/emicklei/go-restful@v2.9.5+incompatible#Container) 이다.
즉, Container는 HTTP 요청을 다중화하기 위한 [WebService](https://pkg.go.dev/github.com/emicklei/go-restful@v2.9.5+incompatible#WebService) collection을
보유하며 WebService에는 root path가 지정되며 최종적으로 path와 method가 [Route](https://pkg.go.dev/github.com/emicklei/go-restful@v2.9.5+incompatible#Route) 에 의해 바인딩된다.
(Container -> WebService -> Route)

NonGoRestfulMux
: chain에서 가장 마지막에 호출되는 HTTP handler이다. mux 객체를 래핑하고 exposedPaths를 기록하는 *mux.PathRecorderMux이며 모든 filter와 API가 처리된 후에 불리게 된다.
여기를 통해 다른 서버들이 chain의 다양한 부분에 handler를 추가할 수 있다.

Director
: 해당 path가 GoRestfulContainer에 의해 처리될 수 있는지를 확인하고 호출하며,
만약 처리될 수 없는 요청이면 NonGoRestfulMux를 호출해 정상적으로 응답이 될 수 있도록 유도한다.
이렇게 동작하는 Director에 의해 실패 및 프록시 케이스를 적절히 처리할 수 있게 된다.
`apis`를 gorestful에 등록하면 `/apis` 또는 `/apis/*`가 아닌 모든 요청은 404 응답이 전달되며 다른 모든 것을 포함하는 패턴 등록 시도는 gorestful 제약에 의해 실패하게 된다.
이에 mux가 필요한 것이다. Kubernetes에서는 요청이 gorestful에서 처리할 수 없는 path를 포함할 경우 mux로 위임해서 처리하도록 설계했다.

FullHandlerChain
: HTTP 요청에 응답하는 http.Handler이다. 전체 filter chain가 포함된 상태에서 Director를 호출하게 된다.

HTTP 요청이 전달되었을 때의 호출 순서는 다음과 같다:
*FullHandlerChain -> Director -> {GoRestfulContainer, NonGoRestfulMux}*

생성된 APIServerHandler를 통해 `/`, `/debug/*`, `/metrics/`, `version`을 포함한 여러 path를 GoRestfulContainer와 NonGoRestfulMux에 추가한다.
또한 logs 관련 라우팅을 지원하는지 확인 후 `/logs` path를 추가한다.

```shell
❯ kubectl get pods -A -v 6
GET https://192.168.49.2:8443/api/v1/pods?limit=500 200 OK in 15 milliseconds

❯ kubectl get deployments -A -v 6
GET https://192.168.49.2:8443/apis/apps/v1/deployments?limit=500 200 OK in 12 milliseconds
```
{: .nolineno}
*: pods와 deployments의 URL 확인*

Kubernetes는 처음 pods와 같은 리소스를 만들 때에는 `/api`이 root path로 붙도록 설계되었고 이후 추가되는 deployments와 같은 리소스들은 `/apis`를 root path로 시작하도록 설계하고 있어서
이러한 리소스들을 *Line:35*, *Line:60*의 `InstallLegacyAPI()` & `InstallAPIs()`에서 각각 바인딩해주고 있다.

`InstallLegacyAPI()`와 `InstallAPIs()`는 [RESTStorage](https://pkg.go.dev/k8s.io/apiserver/pkg/registry/rest#Storage) 를 얻는 방식의 차이가 조금 있고 내부 동작은 거의 비슷하다.
`InstallAPIs()`를 살펴보자.

```go
func (m *Instance) InstallAPIs(apiResourceConfigSource serverstorage.APIResourceConfigSource, restOptionsGetter generic.RESTOptionsGetter, restStorageProviders ...RESTStorageProvider) error {
	apiGroupsInfo := []*genericapiserver.APIGroupInfo{}

	// used later in the loop to filter the served resource by those that have expired.
	resourceExpirationEvaluator, err := genericapiserver.NewResourceExpirationEvaluator(*m.GenericAPIServer.Version)
	if err != nil {
		return err
	}

	for _, restStorageBuilder := range restStorageProviders {
		groupName := restStorageBuilder.GroupName()
		if !apiResourceConfigSource.AnyVersionForGroupEnabled(groupName) {
			klog.V(1).Infof("Skipping disabled API group %q.", groupName)
			continue
		}
		apiGroupInfo, enabled, err := restStorageBuilder.NewRESTStorage(apiResourceConfigSource, restOptionsGetter)
		if err != nil {
			return fmt.Errorf("problem initializing API group %q : %v", groupName, err)
		}
		if !enabled {
			klog.Warningf("API group %q is not enabled, skipping.", groupName)
			continue
		}

		// ...

		apiGroupsInfo = append(apiGroupsInfo, &apiGroupInfo)
	}

	if err := m.GenericAPIServer.InstallAPIGroups(apiGroupsInfo...); err != nil {
		return fmt.Errorf("error in registering group versions: %v", err)
	}
	return nil
}
```

restStorageProviders에 포함돼 있는 [RESTStorageProviders](https://pkg.go.dev/k8s.io/kubernetes/pkg/controlplane#RESTStorageProvider) 를 통해
해당 API Group에 대한 [APIGroupInfo](https://pkg.go.dev/k8s.io/apiserver/pkg/server#APIGroupInfo) 를 빌드하고 모아
*Line:30*의 `InstallAPIGroups(apiGroupInfo...)`를 호출하면서 인자로 전달한다.

RESTStorage
: 특정 리소스나 컬렉션의 정보를 반환하는 동작에 대한 method를 구현할 수 있으며 생성과 변경 및 삭제 전략을 포함하는 인터페이스.
API server에 RESTful API로 등록할 리소스들은 이 인터페이스를 구현해야 한다.

RESTStorageProviders
: RESTStorage를 위한 Factory type이다.
`GroupName()` method로 `/apis/apps`, `/apis/cerificates.k8s.io`와 같은 형태의 groupName을 가져올 수 있고,
`NewRESTStorage()` method로 해당 API Group 내의 RESTStorage를 포함하고 있는 정보인 [APIGroupInfo](https://pkg.go.dev/k8s.io/apiserver/pkg/server#APIGroupInfo) 가져올 수 있다.

APIGroupInfo
: 해당 API Group에 대한 정보를 포함한다.
Group 내의 리소스들에 대한 정보가 들어있는 VersionedResourcesStorageMap을 가지고 있다.

### Resource Handler

*InstallAPIGroups -> installAPIResources -> InstallREST -> registerResourceHandler*까지 도달하면서
apiGroupInfo에 포함된 RESTStorage를 통해 해당 리소스가 지원하는 작업들을 Action으로 저장하고 리소스 경로(*e.g., /api/apiVersion/resource*)에서 표준 REST 동사(GET, PUT, POST 및 DELETE)로 처리될 수 있도록 한다.
예를 들어 create에 해당하는 Action operation은 POST, update에 해당하는 Action operation은 PUT이다.

```go
switch action.Verb {
case "GET": // Get a resource.
   var handler restful.RouteFunction
   if isGetterWithOptions {
     handler = restfulGetResourceWithOptions(getterWithOptions, reqScope, isSubresource)
   } else {
     handler = restfulGetResource(getter, reqScope)
   }

   if needOverride {
     // need change the reported verb
     handler = metrics.InstrumentRouteFunc(verbOverrider.OverrideMetricsVerb(action.Verb), group, version, resource, subresource, requestScope, metrics.APIServerComponent, deprecated, removedRelease, handler)
   } else {
     handler = metrics.InstrumentRouteFunc(action.Verb, group, version, resource, subresource, requestScope, metrics.APIServerComponent, deprecated, removedRelease, handler)
   }
   handler = utilwarning.AddWarningsHandler(handler, warnings)

   doc := "read the specified " + kind
   if isSubresource {
     doc = "read " + subresource + " of the specified " + kind
   }
   route := ws.GET(action.Path).To(handler).
     Doc(doc).
     Param(ws.QueryParameter("pretty", "If 'true', then the output is pretty printed.")).
     Operation("read"+namespaced+kind+strings.Title(subresource)+operationSuffix).
     Produces(append(storageMeta.ProducesMIMETypes(action.Verb), mediaTypes...)...).
     Returns(http.StatusOK, "OK", producedObject).
     Writes(producedObject)
   if isGetterWithOptions {
     if err := AddObjectParams(ws, route, versionedGetOptions); err != nil {
       return nil, nil, err
     }
   }
   addParams(route, action.Params)
   routes = append(routes, route)
```

마지막으로 Action 배열을 차례로 순회하여 각 operation에 handler method를 추가하여 Route에 등록하고, 생성된 Route를 WebService에 등록한다.
모든 Route를 등록한 WebService는 최종적으로 GoRestfulContainer에 등록되어 라우팅된다.
이 작업은 앞서 언급했던 go-restful design pattern과 완벽하게 일치한다.

Container
: 여기서는 APIServerHandlers에 포함된 GoRestfulContainer를 말한다.
HTTP 요청을 디스패치하기 위한 WebService 및 [http.ServeMux](https://pkg.go.dev/net/http#ServeMux) 컬렉션을 가지고 있다.
요청은 [RouteSelector](https://pkg.go.dev/github.com/emicklei/go-restful@v2.9.5+incompatible#RouteSelector) 를 사용해 WebServices의 경로에 추가로 디스패치된다.

ServeMux
: HTTP 요청 Multiplexer이다.
각 수신 요청의 URL을 등록된 패턴의 목록과 일치시켜 URL과 가장 근접한 패턴에 대한 Handler를 호출한다.

RouteSelector
: HTTP 요청이 주어지면 가장 일치하는 최적의 path를 찾는다.
선택적으로 PathProcessor 인터페이스를 구현하여 path가 선택된 후 path parameter도 추출할 수 있다.

WebService
: Route 컬렉션을 가지고 있다.

Route
: HTTP method + URL Path, Comsumers의 조합을 [RouteFunction](https://pkg.go.dev/github.com/emicklei/go-restful@v2.9.5+incompatible#RouteFunction) 에 바인딩한다.

RouteFunction
: Route에 바인딩할 수 있는 function의 signature를 선언한다.

### Execution the final RUN method

이렇게 server는 `CreateServerChain()` method를 통해 생성된다.
이제 다시 처음으로 돌아가 `NewAPIServerCommand()` method 내부에서 실행되는 `Run()` method를 보자.

```go
func Run(completeOptions completedServerRunOptions, stopCh <-chan struct{}) error {
	// ...

	server, err := CreateServerChain(completeOptions, stopCh)
	if err != nil {
		return err
	}

	prepared, err := server.PrepareRun()
	if err != nil {
		return err
	}

	return prepared.Run(stopCh)
}
```

Server를 시작하기 전에 `PrepareRun()` method에서 `/healthz` path 등록을 완료하고 Kubernetes API의 모든 세부 정보 및 사양을 포함한 OpenAPI 라우팅 등록 또한 완료한다.
이후 preparedGenericAPIServer의 `Run()` method를 호출해 `NonBlockingRun()` method를 통해 secure HTTP server를 시작한다.

## Watch Server Creation

Watch Server가 생성되는 과정을 정리하고 마무리해야겠다. registerResourceHandler에서 action.Verb가 LIST일 경우를 살펴보자.

```go
case "LIST": // List all resources of a kind.
  // ...
  handler := metrics.InstrumentRouteFunc(action.Verb, group, version, resource, subresource, requestScope, metrics.APIServerComponent, deprecated, removedRelease, restfulListResource(lister, watcher, reqScope, false, a.minRequestTimeout))
  handler = utilwarning.AddWarningsHandler(handler, warnings)
  route := ws.GET(action.Path).To(handler).
    Doc(doc).
    Param(ws.QueryParameter("pretty", "If 'true', then the output is pretty printed.")).
    Operation("list"+namespaced+kind+strings.Title(subresource)+operationSuffix).
    Produces(append(storageMeta.ProducesMIMETypes(action.Verb), allMediaTypes...)...).
    Returns(http.StatusOK, "OK", versionedList).
    Writes(versionedList)
  // ...
  routes = append(routes, route)
```
Handler를 생성할 때 `restfulListResource()` 함수를 호출하면서 watcher를 넘겨주는 것을 볼 수 있다.
이 watcher는 `watcher, isWatcher := storage.(rest.Watcher)`에서 가져오는 것으로 해당 리소스의 RESTStorage에 이미 등록된 Watcher 작업 그 자체이다.
*restfulListResource -> ListResource -> serveWatch* 순으로 호출되면서 리소스에 알맞는 watch 응답을 성공적으로 제공할 수 있게 되는 것이다.

```go
func serveWatch(watcher watch.Interface, scope *RequestScope, mediaTypeOptions negotiation.MediaTypeOptions, req *http.Request, w http.ResponseWriter, timeout time.Duration) {
	defer watcher.Stop()

	// ...

	server := &WatchServer{
		Watching: watcher,
		Scope:    scope,

		UseTextFraming:  useTextFraming,
		MediaType:       mediaType,
		Framer:          framer,
		Encoder:         encoder,
		EmbeddedEncoder: embeddedEncoder,

		Fixup: func(obj runtime.Object) runtime.Object {
			result, err := transformObject(ctx, obj, options, mediaTypeOptions, scope, req)
			if err != nil {
				utilruntime.HandleError(fmt.Errorf("failed to transform object %v: %v", reflect.TypeOf(obj), err))
				return obj
			}
			// When we are transformed to a table, use the table options as the state for whether we
			// should print headers - on watch, we only want to print table headers on the first object
			// and omit them on subsequent events.
			if tableOptions, ok := options.(*metav1.TableOptions); ok {
				tableOptions.NoHeaders = true
			}
			return result
		},

		TimeoutFactory: &realTimeoutFactory{timeout},
	}

	server.ServeHTTP(w, req)
}
```

`serveWatch()`에서 WatchServer는 전달된 watcher를 가지고 생성되며
[Kubernetes의 Watch event는 어떻게 동작하는가](http://blog.wqlee.com/posts/what-is-a-kubernetes-watch-event/) 의 마지막쯔음에 보았던
ServeHTTP method가 실행되면서 내부 로직을 통해 일련의 인코딩된 이벤트를 클라이언트에게 제공하게 된다.

## Conclusion

생략을 나름대로 최대한 했음에도 불구하고 굉장히 정교하고 복잡한 핵심 코어인지라 API server 코드 분석이 매우 길어졌다.
부족한 부분들이 아쉽기도 한데 이쯤 정리하는 선에서 일단은 만족하기로 했다.
이 포스팅과 관련해 추가적인 수정 사항이나 보충할 내용이 생긴다면 그 때 업데이트할 예정이다.

이번 분석을 통해 API server에 요청된 watch event가 어떠한 방식으로 Watch Server에 전달이 되는지 알게 되었다.
수많은 클라이언트가 API server와 watch event를 위해 connection을 맺고 있다면 어느 정도 부하가 생길 수도 있다고 다들 이야기를 하고 있으며 공감도 된다.
더 효율적인 접근을 위해 Kubernetes에서는 Informer를 제공해주고 있다고 하는데
다음으로는 Informer란 어떤 것인지, 그리고 어떠한 아키텍처를 통해 위와 같은 문제를 해결했는지 정리해보자.

<div style="text-align: center; font-weight: bold; margin-top: 100px; margin-bottom: 50px">끝.</div>

---
[^1]: https://github.com/spf13/cobra
[^2]: https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/
