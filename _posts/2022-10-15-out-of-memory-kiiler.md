---
title: Linux OOM Killer
author: wq
name: Wongyu Lee
link: https://github.com/kyu21
date: 2022-08-20 20:00:00 +0900
categories: [linux]
tags: [os]
render_with_liquid: false
---

# OOM Killer?

[linux OOM Killer](http://linux-mm.org/OOM_Killer)

minikube

systemd / cgroup v2

## QoS

### Guaranteed

```shell
uid: 3da16733_7851_49a0_a806_dd8a38f38ad3
containerID: 9e20d88aa83a55eacdeeb366424c7b0c5c7734213d4d9aeb4ac5e6cd4c8facfe

docker@minikube:/sys/fs/cgroup/kubepods.slice/kubepods-pod3da16733_7851_49a0_a806_dd8a38f38ad3.slice/docker-9e20d88aa83a55eacdeeb366424c7b0c5c7734213d4d9aeb4ac5e6cd4c8facfe.scope$ grep "" memory.*
memory.current:196608
memory.high:max
memory.low:0
memory.max:209715200
memory.min:0

docker@minikube:/sys/fs/cgroup/kubepods.slice/kubepods-pod3da16733_7851_49a0_a806_dd8a38f38ad3.slice/docker-9e20d88aa83a55eacdeeb366424c7b0c5c7734213d4d9aeb4ac5e6cd4c8facfe.scope$ cat cgroup.procs
44318

docker@minikube:/sys/fs/cgroup/kubepods.slice/kubepods-pod3da16733_7851_49a0_a806_dd8a38f38ad3.slice/docker-9e20d88aa83a55eacdeeb366424c7b0c5c7734213d4d9aeb4ac5e6cd4c8facfe.scope$ grep "" /proc/44318/oom_*
/proc/44318/oom_adj:-16
/proc/44318/oom_score:2
/proc/44318/oom_score_adj:-997
```

### BestEffort

```shell
uid: fd217f17-4ffa-41e6-882c-ced81693e3f4
containerID: //5226c0c3615eb8a4de58d7f12dbb64cd68708457b06925d635bf4d84e7bdb679

docker@minikube:/sys/fs/cgroup/kubepods.slice/kubepods-besteffort.slice/kubepods-besteffort-podfd217f17_4ffa_41e6_882c_ced81693e3f4.slice/docker-5226c0c3615eb8a4de58d7f12dbb64cd68708457b06925d635bf4d84e7bdb679.scope$ grep "" memory.*
memory.current:204800
memory.high:max
memory.low:0
memory.max:max
memory.min:0

docker@minikube:/sys/fs/cgroup/kubepods.slice/kubepods-besteffort.slice/kubepods-besteffort-podfd217f17_4ffa_41e6_882c_ced81693e3f4.slice/docker-5226c0c3615eb8a4de58d7f12dbb64cd68708457b06925d635bf4d84e7bdb679.scope$ cat cgroup.procs
42981

docker@minikube:/sys/fs/cgroup/kubepods.slice/kubepods-besteffort.slice/kubepods-besteffort-podfd217f17_4ffa_41e6_882c_ced81693e3f4.slice/docker-5226c0c3615eb8a4de58d7f12dbb64cd68708457b06925d635bf4d84e7bdb679.scope$ grep "" /proc/42981/oom_*
/proc/42981/oom_adj:15
/proc/42981/oom_score:1332
/proc/42981/oom_score_adj:1000
```

### Burstable

```shell
uid: 87431cda-dfb8-4686-a593-4bee7a2596fc
containerID: docker://737d43b63cc68d5b8206adbde941f933f8d374092e5801bb5a875d942fced51f

docker@minikube:/sys/fs/cgroup/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod87431cda_dfb8_4686_a593_4bee7a2596fc.slice/docker-737d43b63cc68d5b8206adbde941f933f8d374092e5801bb5a875d942fced51f.scope$ grep "" memory.*
memory.current:208896
memory.high:max
memory.low:0
memory.max:209715200
memory.min:0

docker@minikube:/sys/fs/cgroup/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod87431cda_dfb8_4686_a593_4bee7a2596fc.slice/docker-737d43b63cc68d5b8206adbde941f933f8d374092e5801bb5a875d942fced51f.scope$ cat cgroup.procs
43918

docker@minikube:/sys/fs/cgroup/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod87431cda_dfb8_4686_a593_4bee7a2596fc.slice/docker-737d43b63cc68d5b8206adbde941f933f8d374092e5801bb5a875d942fced51f.scope$ grep "" /proc/43918/oom_*
/proc/43918/oom_adj:15
/proc/43918/oom_score:1328
/proc/43918/oom_score_adj:994
```

| QoS        | Memory Request | Memory Limit | memory.min | memory.max | memory.low | memory.high | oom_adj | oom_score | oom_score_adj |
|------------|----------------|--------------|------------|------------|------------|-------------|---------|-----------|---------------|
| BestEffort | -              | -            | 0          | max        | 0          | max         | 15      | 1332      | 1000          |
| Burstable  | 100Mi          | 200Mi        | 0          | 209715200  | 0          | max         | 15      | 1328      | 994           |
| Guaranteed | 200Mi          | 200Mi        | 0          | 209715200  | 0          | max         | -16     | 2         | -997          |

- memory request의 값 자체는 cgorp의 memory 설정과는 관계가 없다. QoS가 bustable일 경우 oom_score_adj를 결정할 때 사용된다.
- memory limit의 값이 memory.max에 반영된다.

## oom_score

### kubelet에서의 oom_score_adj

[GetContainerOOMScoreAdjust](https://github.com/kubernetes/kubernetes/blob/master/pkg/kubelet/qos/policy.go#L40)

kubelet은 pod의 QoS를 기반으로 각 컨테이너에 대한 oom_score_adj를 설정한다.

| QoS        | oom_score_adj                                                                     |
|------------|-----------------------------------------------------------------------------------|
| BestEffort | 1000                                                                              |
| Burstable  | min(max(3, 1000 - (1000 * memoryRequestBytes) / machineMemoryCapacityBytes), 999) |
| Guaranteed | -997                                                                              |
| Kubelet    | -999                                                                              |

### linux에서의 oom_score

[oom_kill.c](https://github.com/torvalds/linux/blob/master/mm/oom_kill.c)

[Another OOM killer](https://lwn.net/Articles/391222/)

out_of_memory -> select_bad_process -> oom_evaluate_task -> oom_badness 순으로 호출되며 죽일 후보를 결정하기 위한 점수를 계산한다.
목표는 후속 oom 실패를 방지하기 위해 가장 많은 메모리를 사용 중인 프로세스에게 가장 높은 점수를 부여하는 것이다.

badness score(point)의 기준은 각 프로세스의 RSS, pagetable 및 swap 공간이 사용하는 RAM의 비율이다.
이 계산은 결과적으로 퍼센트*10의 숫자를 반환한다.
즉, 사용 가능한 메모리의 모든 바이트를 사용하는 프로세스의 점수는 1,000이고 메모리를 전혀 사용하지 않는 프로세스의 점수는 0이 된다.
만약 root 소유의 프로세스라면 user 소유의 프로세스보다 조금 더 가치가 있다는 개념에 따라 30을 뺀다.
이러한 결과에 -1,000 ~ 1,000의 범위를 가진 oom_score_adj를 더한 값이 최종 점수가 된다.
-1,000으로 설정하면 oom kill이 완전히 비활성화되는 반면 1,000으로 설정하면 oom killer에 의해 선택될 확률이 매우 높아진다.
oom_adj 값은 사용되지 않는다.

## cgroup out of memory

cgroup의 메모리 사용량이 memory.max를 초과하면 oom kill이 동작하게 된다.

// 내용 추가 필요한가

## node pressure eviction (kubelet)

[Node-pressure-eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)

// 링크로 대체 가능

## node out of memory (oom_killer)

kubelet의 메모리 회수가 가능하기 이전에 노드에 메모리 부족(out of memory, OOM) 이벤트가 발생하면,
노드는 oom_killer에 의존한다.

// 내용 추가할까..
