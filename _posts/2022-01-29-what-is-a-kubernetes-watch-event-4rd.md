---
title: Event란건 어쩌면 어려운게 아닐까
author: wq
name: Wongyu Lee
link: https://github.com/kyu21
date: 2022-01-29 20:00:00 +0900
categories: [kubernetes, analyze]
tags: [kubernetes, watch, event]
render_with_liquid: false
---

## Kubernetes Architecture

Kubernetes는 microservice 기반의 아키텍처가 적용되어 있다.
Control Plane, Worker Node의 컴포넌트들은 개별 서비스로 구현된다.
특히 Control Plane은 Event와 함께 느슨하게 결합된 구성 요소의 원칙을 많이 접목시켜 사용하고 있다.
이러한 아키텍처의 주요 이점은 개발 및 배포의 유연성과 독립성, 수평 확장성 및 장애 허용이라고 할 수 있겠다.
즉 etcd, scheduler, api server 등과 같은 중요한 프로세스를 두 개 이상의 인스턴스로 실행한다.

이러한 개별 서비스들은 Control Loop(관찰 -> 분석 -> 행위)로 모델링되면서 오직 api server와 통신을 한다.
즉, 서로 직접 통신을 하지 않는다.

이러한 특성 탓에 Kubernetes는 Microservice 간의 흐름이 중앙 조정자없이 구성되는 event-driven architecture라고도 한다.

## What is an Event?

이벤트란건 단순히 일어난 불변의 사실이다.
Kubernetes control plane에서 여러 구성 요소들이 API server를 통해 Kubernetes의 개체들을 변경하며, 이러한 변경에 대한 작업은 결국 event로 이어진다.

#### Producers and/or consumers

모든 프로세스(또는 컨트롤러)는 이벤트의 생산하는 producer일 수도 있고, 이벤트를 소비하는 consumer일 수도 있다.

소비자는 API server에서 이벤트를 수신하려는 개체를 지정한다.
이를 Kubernetes에서는 listWatch라고 한다.
소비자가 관찰을 시작할 때엔 관련된 리소스의 모든 리스트를 나열한 다음
특정 리소스 버전 이후의 이벤트에 대해서만 watch 모드로 전환하여 API server의 부하를 줄이도록 설계가 되어 있다.

소비자와 생산자는 대기열 등의 구조들에 의해 완전히 분리되어 있으며 서로에 대해 직접적으로 알 필요가 전혀 없다.
이 구조는 전체 시스템을 아주 쉽게 확장 가능하도록 하는 가장 큰 요인이 될 것이다.

따라서 이벤트가 생산자에서 소비자로 전파되는데 시간이 약간의 걸리기는 할테지만(거의 실시간), 설계 상 완전히 비동기적이고 궁극적으로는 일관된 플랫폼이다.

## Kubernetes core design concept

Event의 상태 변경을 감지하는 2가지 원칙적인 옵션이 존재한다.

Edge-driven triggers
: 상태 변경이 발생하는 시점(e.g., 존재하지 않던 Pod가 실행됨)에 handler가 트리거된다.

Level-driven triggers
: 상태는 주기적으로 확인되며 특정 조건이 충족(e.g., Pod가 실행 중)되면 handler가 트리거 된다.  Polling의 한 형태라고도 할 수 있다.

Kubernetes 개체의 변경은 event로 이어진다고 했다.
프로세스(또는 컨트롤러)의 맥락에서 보자면 발생되는 이벤트에 언제 어떻게 반응해야할지, 구현에 대한 고민이 생길 수 있다.
Kubernetes에서는 이를 어떻게 해결했는지 크게 두 주제를 살펴보자.

#### Level Triggering and Soft Reconciliation

1. Edge-driven logic
   - 네트워킹이 끊어지면 event가 손실될 수 있고, controller 자체에 버그가 존재하거나 일부 외부 cloud API가 다운되는 경우도 있어서 누락된 event에 대해 잘 대처하지 못한다.
   - replicaSet controller가 pod 조욜 시에만 해당 pod를 교체한다고 가정했을 때, 누락된 event는 전체 상태를 조정하지 않기 때문에 항상 더 적은 수의 pod가 실행될 수 있게 된다.
2. Edge-triggered, level-driven logic
   - cluster의 최신 상태를 기반으로 로직을 구현하기 때문에 다른 이벤트가 수신될 때 1.에서 발생될 수 있는 문제를 복구한다.
   - replicaSet controller의 경우 항상 지정된 replicas를 클러스터에서 실행 중인 pod와 비교하며 event가 손실되더라고 다음 pod update가 수신될 때 누락된 모든 pod를 교체한다.
3. Reconciliation with resync
   - 2.에 지속적인 동기화를 추가한다.
   - 일반적인 edge-driven trigger의 문제를 감안할 때, kubernetes controller는 3.의 전략으로 돌아간다.

컨트롤러는 기본적으로 stateless하다.
이벤트 기반 설계는 이벤트 소싱 개념과 유사하게 컨트롤러가 (재)시작될 때 모든 (적절한) 이벤트를 replay한다.
Kubernetes의 컨트롤러는 Edge-Driven Triggers가 아닌 Level-Driven Triggers로 동작하도록 설계되었기 때문에
네트워크 이슈나 컨트롤러 다운타임 동안 이벤트를 놓치게 되면 다음 동기화(Soft Reconciliation)가 있을 때 해당 개체애 대한 완전한 상태를 수신할 수 있게 된다.

#### Optimistic Concurrency

Etcd에 정보가 저장이 될 때 Kubernetes의 구조 상 동일 정보가 두 번 이상 전달될 수 있고
컨트롤러가 서로 간에 직접 통신하지 않기 때문에 상태가 변경될 때 경쟁 조건이 발생할 가능성이 있다.
낙관적 동시성을 통해 다른 컨트롤러에서도 동일한 개체에 쓰기 작업을 할 경우 충돌이 발생되며 각자 컨트롤러에서 충돌에 대한 후처리를 해야만 한다.
Kubernetes는 낙관적 동시성을 단조적으로 증가하는 resourceVersion 기반으로 깔끔하게 구현해뒀다.

## Watch Event in kubeAPIServer

Kubernetes에서의 Watch Event는 아래와 같은 흐름으로 전달이 된다.

| etcd watcher | -> watch cache | -> cacheWatcher(s) |
| ------------ | -------------- | ------------------ |
| Get events from etcd | Implement Store interface | Serve events to clients |

Cacher와 Etcd storage는 API server가 시작될 때 지원해야할 리소스들의 정보에 따라 API가 셋업되면서 만들어진다.
Cacher 내부에 포함된 Reflector에서 호출될 ListAndWatch 메서드에서의 watch 작업은  etcd watcher가 포함된 etcd storage의 watch 메서드를 호출한다.

![cacher](/images/cacher.png)
_그림 1. Cacher_

etcd watcher는 etcd v3 client session을 제공하고 관리하는 Client의 Watch를 통해(아마 여기에서 gRPC를 통해 etcd와 bidirection stream으로 이벤트를 전달받을 것 이다.)
WatchResponse를 가져와 파싱하고, watchChan.incomingEventChan으로 전달한다.

![etcd watcher](/images/etct-watcher.png)
_그림 2. Etcd storage_

processEvent를 톨해 etcd watcher의 이벤트를 incomingEventChan으로 받아 처리하고 그 결과를 resultChan으로 보낸다.
이 결과는 곧 Cacher 내부의 reflector가 수행하는 watch로 전달되어 watchCache가 받아 processEvent를 통해 이벤트를 전달하게 되고
Cacher의 Watchers에 등록되어 있던 모든 cacheWatcher들에게 해당 이벤트를 최종적으로 전달해준다.

<div style="text-align: center; font-weight: bold; margin-top: 100px; margin-bottom: 50px">끝.</div>
