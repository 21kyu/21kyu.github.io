---
title: Control groups & Kubernetes
author: wq
name: Wongyu Lee
link: https://github.com/kyu21
date: 2022-08-20 20:00:00 +0900
categories: [linux]
tags: [os]
render_with_liquid: false
---

# Control groups?

보통 cgroups라고 불리는 control gorups는 프로세스가 사용하는 다양한 유형의 리소스를 제한하고 모니터링할 수 있는,
계층적 그룹으로 구성된 Linux kernel 기능이다.
커널의 cgroup interface는 `cgroupfs`라는 pseudo-filesystem을 통해 제공된다.

## Terminology

cgorup
: cgorup은 cgroup filesystem을 통해 정의된 제한 또는 매개변수 집합에 바인딩된 프로세스들을 말한다.


subsystem
: subsystem은 group에서 프로세스의 동작을 제어하는 커널의 구성 요소이다.
cgorup에 허용될 CPU 및 메모리 양에 대한 제한, cgroup에서 사용하는 CPU 시간에 대한 측정, 프로세스 실행 중지 및 재개와 같은 작업을 수행할 수 있는 다양한 하위 시스템이 구현되어 있다.
subsystem은 리소스 컨트롤러(또는 간단히 컨트롤러)라고도 한다.

hierarchy
: controller의 cgorups는 계층 구조로 정렬된다.
이 계층 구조는 cgroup filesystem 내에서 하위 디렉토리를 생성, 제거 작업을 통해 정의된다.
계층의 각 수준에서 속성(e.g., 제한)을 정의할 수 있다.
cgorup에서 제공하는 제한, 제어 및 측정은 보통 속성이 정의된 cgorup 아래의 하위 계층 전체에 영향을 미친다.
따라서 계층의 더 높은 수준에서 cgorup에 설정된 제한값은 하위 cgorups가 초과할 수 없게 된다.

# CGROUPS VERSION 1

cgroups v1에서의 각 controller는 시스템 내 프로세스에 대한 자체 계층 구조를 제공하는 별도의 cgorup filesystem으로 마운트될 수 있다.
동일한 cgorup filesystem에 대해 여러(또는 모든) cgorup v1 controllers를 같이 마운트하는 것도 가능하다.
즉, 같이 마운트된 controllers는 동일한 계층적 프로세스 조직을 관리하게 된다.

마운트된 각 계층에 대해 디렉토리 트리는 control group 계층을 미러링한다.
각 control group은 디렉토리로 표시되며 각 하위 cgroups는 하위 디렉토리로 표시된다.
예를 들어서, /user/joe/1.session은 /user의 자식인 cgorup joe의 자식인 control group 1.session을 나타낸다.
각 cgorup 디렉토리 아래에는 리소스 제한과 몇 가지 일반적인 cgroup 속성을 반영하여 읽거나 쓸 수 있는 파일들이 있다.

## Tasks (threads) versus processes

cgorups v1에서는 프로세스와 태스크를 구분한다.
프로세스는 여러 (사용자 공간 관점에서 대개 스레드로 불리우는) 태스크로 구성될 수 있다.
cgroups v1에서는 프로세스에 있는 스레드의 cgroup 구성원을 독립적으로 조작할 수 있다.

cgroups v1이 스레드를 여러 cgroup으로 분할하는 기능으로 인해 상황에 따라 문제가 발생되곤 했는데,
프로세스의 모든 스레드가 단일 주소 공간을 공유하기 때문에 memory 컨트롤러는 큰 의미가 없다.
이러한 문제로 인해 프로세스에서 스레드의 cgorup 구성원을 독립적으로 조작하는 기능은 초기 cgorups v2 구현에서 제거되었으며
추후 더 제한된 형태로 복원되긴 했다.

## Mounting v1 controllers

cgroups를 사용하려면 CONFIG_CGROUP 옵션으로 빌드된 커널이 필요하다.
또한 각 v1 컨트롤러에는 해당 컨트롤러를 사용하기 위해 설정해야하는 관련된 구성 옵션이 존재한다.

v1 컨트롤러 사용을 위해 cgroup filesystem을 마운트하자.
보통 sys/fs/cgroup에 마운트된 tmpfs filesystem 아래에 위치한다.
따라서 다음과 같이 cpu 컨트롤러를 마운트할 수 있다.

```shell
mount -t cgroup -o cpu none /sys/fs/cgroup/cpu
```

동일한 계층 구조에 대해 여러 컨트롤러를 공동으로 마운트할 수도 있다.
다음의 명령어를 사용하면 cpu 및 cpuacct 컨트롤러는 단일 계층 구조에 함께 마운트된다.

```shell
mount -t cgroup -o cpu,cpuacct none /sys/fs/cgroup/cpu,cpuacct
```

컨트롤러를 함께 마운트하면 프로세스가 같이 마운트된 모든 컨트롤러에 대해 동일한 cgroup에 있다는 효과를 볼 수 있다.
또한 동일한 계층 구조에 대해 모든 v1 컨트롤러를 함께 마운트할 수 있다.

```shell
mount -t cgroup -o all cgroup /sys/fs/cgroup
```

여러 cgroup 계층에 대해 동일한 컨트롤러를 마운트할 수 없다.
한 계층에서 cpu 및 cpuacct 컨트롤러를 모두 마운트하고 다른 계층에 대해 cpu 컨트롤러만 따로 마운트할 수 없다.
lssubsys를 사용해 각 subsystem이 마운트된 경로를 확인할 수 있다.

```shell
➜  ~ lssubsys -am                                                                
cpu,cpuset,memory /cgroup/cpu_and_mem
cpuacct
blkio
devices
freezer
net_cls
perf_event
net_prio
hugetlb
pids
rdma
misc
```

## Unmounting v1 controllers

마운트된 cgroup filesystem은 umount 커맨드를 사용해 마운트 해제할 수 있다.

```shell
umount /sys/fs/cgroup/pids
```

cgroup filesystem은 사용 중이 아닌 경우, 즉 자식 cgroup이 없는 경우에만 마운트 해제된다.
그렇지 않은 경우의 umount 커맨드의 효과는 해당 마운트를 보이지 않게만 만든다.
따라서 마운트가 실제로 해제되었는지 확인하려면 먼저 모든 하위 cgroup을 제거해야만 한다.

## cgroups v1 controllers

cpu
: cgorups는 시스템이 사용 중일 때 최소 "CPU 공유" 수를 보장할 수 있다.
CPU가 사용 중이 아닌 경우 cgorups의 CPU 사용량을 제한하지 않는다.
Linux 3.2에서 이 컨트롤러는 CPU "대역폭" 제어를 제공하도록 확장되었다.
커널이 CONFIG_CFS_BANDWITDH로 구성된 경우 각 스케줄링 기간(cgorup 디렉토리의 파일을 통해 정의됨) 내에서
cgroup의 프로세스에 할당된 CPU 시간의 상한을 정의할 수 있다.
이 상한은 CPU에 대한 다른 경쟁이 없는 경우에도 적용된다.

cpuacct
: 프로세스 그룹에 의해 사용되고 있는 CPU 사용량에 대한 관측 정보를 제공한다.

cpuset
: cgroup의 프로세스를 지정된 CPU 및 NUMA 노드 집합에 바인딩하도록 사용할 수 있다.

memory
: cgroup에서 사용하는 프로세스 메모리, 커널 메모리 및 swap의 보고 및 제한을 지원한다.

device
: 특정 프로세스가 (mknod) 장치를 생성하고 읽기 또는 쓰기를 위해 열 수 있는지에 대한 제어를 할 수 있도록 한다.
정책은 허용 및 거부 목록으로 지정할 수 있다.
계층 구조로 적용되므로 새로운 규칙은 대상 또는 상위 cgorup에 대한 기존 규칙을 위반해서는 안된다.

freezer
: cgorup의 모든 프로세스를 일시 중단하고 복원(재개)할 수 있다.
cgorup /A를 프리징하면 하위 프로세스(/A/B) 또한 프리징된다.

net_cls
: cgorup에 의해 생성된 네트워크 패킷에 cgorup에 대해 지정된 classid를 배치한다.
이렇게 배치된 classid들은 방화벽 규칙 및 tc를 사용해 트래픽을 형성하는데 사용할 수 있다.
이것은 cgroup에 도착하는 트래픽이 아니라 cgroup을 나가는 패킷에만 적용된다.

blkio
: 스토리지 계층의 리프(leaf) 노드 및 중간(intermediate) 노드에 대한 스로틀링 및 상한(upper limits)의 형태로
IO 제어를 적용해 지정된 block devices에 대한 접근을 제어하고 제한한다.
CFQ를 사용하는 리프 노드에 적용되는 디스크의 비례 가중치(proportional-weight) 시간 기반 분할(time-based division)과
device의 상위 I/O 속도 제한을 조절하는 두 가지의 정책을 사용할 수 있다.

perf_event
: cgroup으로 그룹화된 프로세스들을 모니터링할 수 있다.

net_prio
: cgroups에 대해 네트워크 인터페이스별로 우선 순위를 지정해줄 수 있다.

hugetlb
: cgroups에 의한 huge page 사용 제한을 지원한다.

pids
: cgroup(및 그 하위 자식들)에서 생성될 수 있는 프로세스의 수를 제한한다.

rdma
: cgroup당 RDMA/IB 특정 리소스의 사용을 제한할 수 있다.

## Creating cgroups and moving processes

새로운 cgroup은 cgroup filesystem에 디렉토리를 만들면 생성된다.
cgroup filesystem은 /sys/fs/cgroup에 위치해있다.

```shell
➜  ~ cd /sys/fs/cgroup 
➜  cgroup ls
cgroup.controllers      cgroup.subtree_control  cpu.stat             io.cost.qos       memory.pressure                sys-kernel-config.mount
cgroup.max.depth        cgroup.threads          dev-hugepages.mount  io.pressure       memory.stat                    sys-kernel-debug.mount
cgroup.max.descendants  cpu.pressure            dev-mqueue.mount     io.prio.class     misc.capacity                  sys-kernel-tracing.mount
cgroup.procs            cpuset.cpus.effective   init.scope           io.stat           proc-sys-fs-binfmt_misc.mount  system.slice
cgroup.stat             cpuset.mems.effective   io.cost.model        memory.numa_stat  sys-fs-fuse-connections.mount  user.slice
```

이 호스트는 `cgroup.controllers`와 같은 파일이 있는걸로 봐선 cgroups v2 파일시스템을 지원한다.
참고로 현재 사용 중인 Linux system이 지원하는 cgroup 버전은 아래 커맨드로 쉽게 확인할 수 있다.

```shell
➜  cpu grep cgroup /proc/filesystems
nodev	cgroup
nodev	cgroup2
```

새 cgroup을 만들어보자.

```shell
➜  cgroup sudo mkdir -p cpu/cg1
➜  cgroup cd cpu   
➜  cpu ls
cg1                     cgroup.procs            cpu.max.burst          cpu.stat         io.prio.class        memory.low        memory.swap.current
cgroup.controllers      cgroup.stat             cpu.pressure           cpu.uclamp.max   io.stat              memory.max        memory.swap.events
cgroup.events           cgroup.subtree_control  cpuset.cpus            cpu.uclamp.min   io.weight            memory.min        memory.swap.high
cgroup.freeze           cgroup.threads          cpuset.cpus.effective  cpu.weight       memory.current       memory.numa_stat  memory.swap.max
cgroup.kill             cgroup.type             cpuset.cpus.partition  cpu.weight.nice  memory.events        memory.oom.group  pids.current
cgroup.max.depth        cpu.idle                cpuset.mems            io.max           memory.events.local  memory.pressure   pids.events
cgroup.max.descendants  cpu.max                 cpuset.mems.effective  io.pressure      memory.high          memory.stat       pids.max
➜  cpu cd cg1   
➜  cg1 ls
cgroup.controllers  cgroup.freeze  cgroup.max.depth        cgroup.procs  cgroup.subtree_control  cgroup.type   cpu.stat     memory.pressure
cgroup.events       cgroup.kill    cgroup.max.descendants  cgroup.stat   cgroup.threads          cpu.pressure  io.pressure
```

현재 사용 중인 shell 프로세스의 PID를 cgroup의 cgroup.procs 파일에 기록하여 해당 cgroup으로 이동할 수 있다.

```shell
➜  echo $$ > cgroup.procs
➜  cat cgroup.procs 
1545167
1545240
➜  ps aux | grep 1545167
root     1545167  0.0  0.0  11544  4304 pts/1    S    23:04   0:00 bash
➜  tty
/dev/pts/1
```

이 파일에는 한 번에 하나의 PID만 기록해야 하며
PID를 cgroup.procs에 쓸 때 프로세스의 모든 스레드는 한 번에 새 cgroup으로 이동한다.
계층 내에서 프로세스는 정확히 하나의 cgroup의 구성원일 수 있다.
프로세스의 PID를 cgroup.procs 파일에 쓰면 이전에 구성원이었던 cgroup에서 자동으로 제거된다.

cgroup.procs 파일을 읽어 cgroup의 구성원이 프로세스 리스트를 얻을 수 있다.
반환된 PID 리스트는 순서가 보장되지 않으며 또한 중복이 없다는 보장도 없다.
(리스트에서 읽는 동안 PID가 재활용될 수 있다.)

cgroups v1에서 개별 스레드는 cgroup 디렉토리의 tasks 파일에
스레드 ID(i.e., clone 및 gettid에 의해 반환된 커널 스레드 ID)를 기록하여 다른 cgroup으로 이동할 수 있다.
이 파일을 읽어 cgroup의 구성원인 스레드 집합을 검색할 수 있다.

## Removing cgroups

cgroup을 제거하려면 먼저 자식 cgroups가 없고 (nonzombie) 프로세스가 없어야 한다.
이런 경우라면 해당 디렉토리 경로 이름을 간단하게 제거할 수 있다.
cgroup 디렉토리의 파일은 제거할 수 없으며 제거할 필요도 없다.

# CGROUPS VERSION 2

cgroups v2에서 마운트된 모든 컨트롤러는 단일 통합 계층 구조(single unified hierarchy)이다.
(다른) 컨트롤러가 v1 및 v2 계층 아래에 동시에 마운트될 수 있지만 v1 및 v2 계층 모두에서 동일한 컨트롤러를 동시에 마운트할 수는 없다.

cgroups v2의 새로운 동작은 다음과 같이 요약된다.

1. Cgroups v2는 모든 컨트롤러가 마운트되는 통합 계층 구조를 제공한다.
2. "내부(Internal)" 프로세스는 허용되지 않는다. 루트 cgroup을 제외하고 프로세스는 리프 노드(하위 cgroup을 포함하지 않는 cgroup)에만 상주할 수 있다.
3. active? cgroups는 cgroup.controllers 및 cgroup.subtree_control 파일을 통해 지정해야 한다.
4. tasks 파일이 제거되었다. 또한 cpuset 컨트롤러에서 사용하는 cgroup.clone_children 파일도 제거되었다.
5. cgroup.events 파일에서 빈 cgroup 알림을 위한 개선된 매커니즘을 제공한다.

## Cgroups v2 unified hierarchy

cgroups v1에서 서로 다른 계층에 대해 각기 다른 컨트롤러를 마운트하는 기능은 애플리케이션 설계에 있어서 큰 유연성을 허용하기 위함이었다.
그러나 실제로 유연성은 예상보다 덜 유용했고 경우에 따라 복잡성이 추가되었다.
따라서 cgroups v2에서는 사용 가능한 모든 컨트롤러는 단일 계층 구조에 마운트된다.
다음과 같은 명령 사용을 통해 cgorups v2 filesystem을 마운트 할 때 컨트롤러를 지정할 필요가 없다(또는 가능하지 않다):

```shell
mount -t cgroup2 none /mnt/cgroup2
```

cgroups v2 컨트롤러는 cgroups v1 계층 구조에서 마운트되지 않은 경우에만 사용할 수 있다.
다르게 말하면 v1 계층 구조와 v2 계층 구조 모두에서 동일한 컨트롤러를 사용할 수 없다.
즉, v1 컨트롤러를 먼저 마운트 해제해야 v2에서 해당 컨트롤러를 사용할 수 있다.
systemd는 기본적으로 일부 v1 컨트롤러를 많이 사용하기 때문에 어떤 경우에서든
선택한 v1 컨트롤러가 비활성화된 상태에서 시스템을 부팅하는 것이 더 간단할 수 있다.
커널 부팅 명령줄에서 `cgroup_no_v1=list` 옵션을 지정하면 된다.
여기에서 `list`는 비활성화할 컨트롤러 이름을 쉼표로 구분한 리스트이며 모든 v1 컨트롤러를 비활성화하려면 all을 사용하면 된다.

최근 많은 시스템에서 [systemd](https://man7.org/linux/man-pages/man1/systemd.1.html)는
부팅 프로세스 동안 `/sys/fs/cgroup/unified`에 `cgroup2` filesystem을 자동으로 마운트한다.

## Cgroups v2 mount options

cgroups v2 filesystem을 마운트할 때 다음 옵션(mount -o)을 지정할 수 있다.

nsdelegate
: cgroup namespaces를 위임 경계?(delegation boundaries)로 취급한다.

memory_localevents
: memory.events는 cgroup 자체에 대한 통계만 표시해야 하며 하위 cgroups에 대해서는 표시하지 않아야 한다(Linux 5.2 이전 동작).
Linux 5.2부터는 memory.events에 하위 cgroups에 대한 통계를 포함하는 것이며 이 마운트 옵션을 사용해 레거시 동작으로 되돌릴 수 있다.
이 옵션은 시슽메 전체에 적용되며 마운트 시 설정하거나 초기 마운트 네임스페이스에서만 다시 마운트를 해 수정할 수 있다.

## Cgroups v2 controllers

다음 컨트롤러들이 cgroups v2에서 지원된다.

cpu
: cgroups v1에서의 cpu와 cpuacct 컨트롤러의 후속 컨트롤러

cpuset
: cgroups v1과 동일

freezer
: cgroups v1과 동일

hugetlb
: cgroups v1과 동일

io
: cgroups v1에서의 blkio 컨트롤러의 후속 컨트롤러

memory
: cgroups v1과 동일

perf_event
: cgroups v1과 동일

pids
: cgroups v1과 동일

rdma
: cgroups v1과 동일

cgroups v1의 `net_cls` 및 `net_prio` 컨트롤러에 직접적으로 대응되는 컨트롤러는 없다.
대신 `iptables`에 지원이 추가되어 cgroups v2 경로 이름에 연결하는 eBPF 필터가 cgroup 기준으로 네트워크 트래픽에 대한 결정을 내릴 수 있다.

v2 `devices` 컨트롤러는 인터페이스 파일을 제공하지 않는다.
대신 eBPF(**BPF_CGROUP_DEVICE**) 프로그램을 cgroup v2에 연결해 디바이스를 제어한다.

## Cgroups v2 subtree control

v2 계층 구조의 각 cgroup에는 다음 두 파일이 포함된다.

cgroup.controllers
: 이 읽기 전용 파일은 해당 cgorup에서 사용 가능한 컨트롤러의 리스트를 보여준다.
이 파일의 내용은 상위 cgorup이 있는 `cgroup.subtree_control` 파일의 내용과 일치한다.

cgroup.subtree_control
: cgroup에서 활성화된 컨트롤러의 리스트이다.
이 파일의 컨트롤러 집합은 해당 cgroup의 `cgroup.controllers`에 있는 집합의 하위 집합이다.
활성? 컨트롤러 세트는 다음 예와 같이 공백으로 구분된 컨트롤러 이름을 포함하는 문자열의
각 이름 앞에 '+'(컨트롤러 활성화) 또는 '-'(컨트롤러 비활성화)을 이 파일에 작성한다.
```shell
echo '+pids -memory' > x/y/cgroup.subtree_control
```
`cgroup.controllers`에 없는 컨트롤러를 활성화하려고 하면 `cgroup.subtree_control` 파일에 쓸 때 **ENOENT** 에러가 발생한다.

`cgroup.subtree_control`의 컨틀롤러 리스트는 해당 `cgroup.controller`의 하위 집합이기 때문에
계층 구조의 한 cgroup에서 비활성화된 컨트롤러는 해당 cgroup 아래의 하위 트리에서 다시 활성화할 수 없다.

cgroup의 `cgroup.subtree_control` 파일은 하위 cgroup에서 실행되는 컨트롤러 세트를 결정한다.
컨트롤러(e.g., pids)가 상위 cgroup의 `cgroup.subtree_control` 파일에 있으면 해당 컨트롤러 인터페이스 파일(e.g., pids.max)이
해당 cgroup의 하위에 자동으로 생성되고 이를 사용하여 하위 cgroup의 리소스를 제어할 수 있다.

## Cgroups v2 "no internal processes" rule

cgroups v2는 "내부 프로세스 없음(no internal processes)"이라 불리는 규칙을 시행한다.
대략 말해보자면 루트 cgroup을 제외하고 프로세스들은 리프 노드(하위 cgroups를 포함하지 않는 cgroup)에만 상주할 수 있음을 의미한다.
이렇게 하면 cgroup A의 구성원인 프로세스와 A의 하위 cgroup에 있는 프로세스 간에 리소스를 분할하는 방법을 결정할 필요가 없다.

# Kubernetes cgroups

쿠버네티스에서는 cgroup을 어떻게 사용할까

## Run

kubernetes 역시 /sys/fs/cgroup을 cgroup의 path로 사용한다.

```shell
➜  ~ ls /sys/fs/cgroup                                
cgroup.controllers      cgroup.stat             cpuset.cpus.effective  dev-mqueue.mount  io.pressure    memory.numa_stat  proc-sys-fs-binfmt_misc.mount  sys-kernel-tracing.mount
cgroup.max.depth        cgroup.subtree_control  cpuset.mems.effective  init.scope        io.prio.class  memory.pressure   sys-fs-fuse-connections.mount  system.slice
cgroup.max.descendants  cgroup.threads          cpu.stat               io.cost.model     io.stat        memory.stat       sys-kernel-config.mount        user.slice
cgroup.procs            cpu.pressure            dev-hugepages.mount    io.cost.qos       kubepods       misc.capacity     sys-kernel-debug.mount

➜  ~ cd /sys/fs/cgroup/kubepods
➜  kubepods ls
besteffort          cgroup.max.descendants  cpu.max                cpuset.mems.effective  hugetlb.2MB.events        io.prio.class        memory.low        memory.swap.current  pids.events
burstable           cgroup.procs            cpu.max.burst          cpu.stat               hugetlb.2MB.events.local  io.stat              memory.max        memory.swap.events   pids.max
cgroup.controllers  cgroup.stat             cpu.pressure           cpu.uclamp.max         hugetlb.2MB.max           io.weight            memory.min        memory.swap.high     rdma.current
cgroup.events       cgroup.subtree_control  cpuset.cpus            cpu.uclamp.min         hugetlb.2MB.rsvd.current  memory.current       memory.numa_stat  memory.swap.max      rdma.max
cgroup.freeze       cgroup.threads          cpuset.cpus.effective  cpu.weight             hugetlb.2MB.rsvd.max      memory.events        memory.oom.group  misc.current
cgroup.kill         cgroup.type             cpuset.cpus.partition  cpu.weight.nice        io.max                    memory.events.local  memory.pressure   misc.max
cgroup.max.depth    cpu.idle                cpuset.mems            hugetlb.2MB.current    io.pressure               memory.high          memory.stat       pids.current
```

/sys/fs/cgroup 바로 아래에 /kubepods 폴더가 존재한다. cgroups v2를 사용하는 시스템이라?
실제로 현재 사용 중인 Linux system은 cgroup2fs type을 사용하는 걸로 나온다.

```shell
➜  kubepods stat /sys/fs/cgroup -f
  File: "/sys/fs/cgroup"
    ID: 0        Namelen: 255     Type: cgroup2fs
Block size: 4096       Fundamental block size: 4096
Blocks: Total: 0          Free: 0          Available: 0
Inodes: Total: 0          Free: 0
```

## BestEffort

새 busybox 파드를 배포해보자.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
spec:
  containers:
  - image: busybox
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNotPresent
    name: busybox
  restartPolicy: Always
```

파드는 다음과 같이 잘 배포되었다.
해당 파드의 uid가 `8fea8dd7-a6d2-4319-936f-d81bf4942cc2`이고
busybox 컨테이너의 id가 `4b96c3c56e7a5b1596ff64c58cfa7fdad52de3017918071fcc3229091b4f0c7e`인 것을 확인할 수 있다.

```shell
➜  Workspace k apply -f busybox.yaml
pod/busybox created
➜  Workspace k get po               
NAME      READY   STATUS    RESTARTS   AGE
busybox   1/1     Running   0          8s
➜  Workspace k get po -o yaml | grep uid
    uid: 8fea8dd7-a6d2-4319-936f-d81bf4942cc2
➜  Workspace k get po -o yaml | grep containerID
    - containerID: containerd://4b96c3c56e7a5b1596ff64c58cfa7fdad52de3017918071fcc3229091b4f0c7e
```

지금 해당 값들을 확인하는 이유는 kubernetes가 이 값들을 사용해 하위 cgroup을 구성하기 때문이다.
cgroups path를 확인해보자.
busybox 파드에 아무런 resources 설정을 하지 않았기 때문에 QoS는 BestEffort이므로 해당 폴더를 확인해보면 된다.

```shell
➜  ~ ls /sys/fs/cgroup/kubepods/besteffort 
cgroup.controllers      cgroup.subtree_control  cpuset.cpus.effective  cpu.weight.nice           io.pressure          memory.low           memory.swap.events  pod8fea8dd7-a6d2-4319-936f-d81bf4942cc2
cgroup.events           cgroup.threads          cpuset.cpus.partition  hugetlb.2MB.current       io.prio.class        memory.max           memory.swap.high    rdma.current
cgroup.freeze           cgroup.type             cpuset.mems            hugetlb.2MB.events        io.stat              memory.min           memory.swap.max     rdma.max
cgroup.kill             cpu.idle                cpuset.mems.effective  hugetlb.2MB.events.local  io.weight            memory.numa_stat     misc.current
cgroup.max.depth        cpu.max                 cpu.stat               hugetlb.2MB.max           memory.current       memory.oom.group     misc.max
cgroup.max.descendants  cpu.max.burst           cpu.uclamp.max         hugetlb.2MB.rsvd.current  memory.events        memory.pressure      pids.current
cgroup.procs            cpu.pressure            cpu.uclamp.min         hugetlb.2MB.rsvd.max      memory.events.local  memory.stat          pids.events
cgroup.stat             cpuset.cpus             cpu.weight             io.max                    memory.high          memory.swap.current  pids.max

➜  ~ ls /sys/fs/cgroup/kubepods/besteffort/pod8fea8dd7-a6d2-4319-936f-d81bf4942cc2
4b96c3c56e7a5b1596ff64c58cfa7fdad52de3017918071fcc3229091b4f0c7e  cgroup.stat             cpuset.cpus.effective  hugetlb.2MB.current       io.stat              memory.numa_stat     misc.max
862ac391170008426a5fb8ea84ebe78093fb78ed6a311eba9d1bce0e102e3752  cgroup.subtree_control  cpuset.cpus.partition  hugetlb.2MB.events        io.weight            memory.oom.group     pids.current
cgroup.controllers                                                cgroup.threads          cpuset.mems            hugetlb.2MB.events.local  memory.current       memory.pressure      pids.events
cgroup.events                                                     cgroup.type             cpuset.mems.effective  hugetlb.2MB.max           memory.events        memory.stat          pids.max
cgroup.freeze                                                     cpu.idle                cpu.stat               hugetlb.2MB.rsvd.current  memory.events.local  memory.swap.current  rdma.current
cgroup.kill                                                       cpu.max                 cpu.uclamp.max         hugetlb.2MB.rsvd.max      memory.high          memory.swap.events   rdma.max
cgroup.max.depth                                                  cpu.max.burst           cpu.uclamp.min         io.max                    memory.low           memory.swap.high
cgroup.max.descendants                                            cpu.pressure            cpu.weight             io.pressure               memory.max           memory.swap.max
cgroup.procs                                                      cpuset.cpus             cpu.weight.nice        io.prio.class             memory.min           misc.current

➜  ~ ls /sys/fs/cgroup/kubepods/besteffort/pod8fea8dd7-a6d2-4319-936f-d81bf4942cc2/4b96c3c56e7a5b1596ff64c58cfa7fdad52de3017918071fcc3229091b4f0c7e
cgroup.controllers      cgroup.stat             cpu.pressure           cpu.uclamp.max            hugetlb.2MB.max           io.weight            memory.min           memory.swap.high  rdma.current
cgroup.events           cgroup.subtree_control  cpuset.cpus            cpu.uclamp.min            hugetlb.2MB.rsvd.current  memory.current       memory.numa_stat     memory.swap.max   rdma.max
cgroup.freeze           cgroup.threads          cpuset.cpus.effective  cpu.weight                hugetlb.2MB.rsvd.max      memory.events        memory.oom.group     misc.current
cgroup.kill             cgroup.type             cpuset.cpus.partition  cpu.weight.nice           io.max                    memory.events.local  memory.pressure      misc.max
cgroup.max.depth        cpu.idle                cpuset.mems            hugetlb.2MB.current       io.pressure               memory.high          memory.stat          pids.current
cgroup.max.descendants  cpu.max                 cpuset.mems.effective  hugetlb.2MB.events        io.prio.class             memory.low           memory.swap.current  pids.events
cgroup.procs            cpu.max.burst           cpu.stat               hugetlb.2MB.events.local  io.stat                   memory.max           memory.swap.events   pids.max
```

busybox 파드의 uid, containerID와 일치하는 cgroup이 kubepods/besteffort/ 아래에 계층 구조로 만들어진걸 확인할 수 있다.
`/sys/fs/cgroup/kubepods/besteffort/pod8fea8dd7-a6d2-4319-936f-d81bf4942cc2` cgroup 아래에 두 개의 cgroup이 있는 것을 볼 수 있는데,
하나(4b96c3..)는 busybox 컨테이너이고 다른 하나(862ac3..)는 해당 파드의 pause 컨테이너이다. 이는 cgroup.procs 파일로 확인 가능하다.

```shell
➜  ~ cat /sys/fs/cgroup/kubepods/besteffort/pod8fea8dd7-a6d2-4319-936f-d81bf4942cc2/862ac391170008426a5fb8ea84ebe78093fb78ed6a311eba9d1bce0e102e3752/cgroup.procs 
616545
➜  ~ ps aux | grep 616545     
65535     616545  0.0  0.0    972     4 ?        Ss   23:06   0:00 /pause

➜  ~ cat /sys/fs/cgroup/kubepods/besteffort/pod8fea8dd7-a6d2-4319-936f-d81bf4942cc2/4b96c3c56e7a5b1596ff64c58cfa7fdad52de3017918071fcc3229091b4f0c7e/cgroup.procs 
616691
➜  ~ ps aux | grep 616691      
root      616691  0.0  0.0   1320     4 ?        Ss   23:07   0:00 sleep 3600
```

cpu, memory 확인
cgroups v1과는 다르게 생겼다..
v1에서는 cpu.shares, cpu.cfs_period_us, cpu.cfs_quota_us로 확인이 됐었는디

```shell
➜  ~ cd /sys/fs/cgroup/kubepods/besteffort/pod8fea8dd7-a6d2-4319-936f-d81bf4942cc2/4b96c3c56e7a5b1596ff64c58cfa7fdad52de3017918071fcc3229091b4f0c7e              
➜  4b96c3c56e7a5b1596ff64c58cfa7fdad52de3017918071fcc3229091b4f0c7e grep "" cpu.*
cpu.idle:0
cpu.max:max 100000
cpu.max.burst:0
cpu.pressure:some avg10=0.00 avg60=0.00 avg300=0.00 total=325
cpu.pressure:full avg10=0.00 avg60=0.00 avg300=0.00 total=325
cpu.stat:usage_usec 11315
cpu.stat:user_usec 0
cpu.stat:system_usec 11315
cpu.stat:nr_periods 0
cpu.stat:nr_throttled 0
cpu.stat:throttled_usec 0
cpu.uclamp.max:max
cpu.uclamp.min:0.00
cpu.weight:1
cpu.weight.nice:19
```

파드 내 resources 스펙이 어떻게 매칭되는지
[ResourceConfigForPod](https://github.com/kubernetes/kubernetes/blob/76277917b9b98bfac79d0e25fe8f45dfc5bec145/pkg/kubelet/cm/helpers_linux.go#L116) 에서 확인해보자.

1. 파드 내 컨테이너들의 리소스 requests, limits를 더한다.
2. cpuRequests / cpuLimits / memoryLimits에 대입한다.
    1. cpuRequests <- requests[cpu]
    2. cpuLimits <- limits[cpu]
    3. memoryLimits <- limits[memory]
3. 2.에서 구한 값들을 CFS 값으로 변경한다.
    1. cpuShares(min: 2, max: 262144) <- MilliCPUToShares(cpuRequests)
    2. cpuQuota <- MulliCPUToQuota(cpuLimits)

결과적으로 해당 값들은 [Resources](https://pkg.go.dev/github.com/opencontainers/runc/libcontainer/configs@v1.1.3#Resources) 타입으로
libcontainer의 Set 메서드의 인자로 전달된다.

cgroups v1의 경우 호출되는 [Set](https://github.com/opencontainers/runc/blob/main/libcontainer/cgroups/fs/cpu.go#L51) 에 의해
다음과 같이 설정된다.

- cpuShares -> cpu.shares
- cpuPeriod -> cpu.cfs_period_us
- cpuQuota -> cpu.cfs_quota_us

cpu.shares
: cgroup의 작업에 사용 가능한 상대적인 cpu 시간을 지정할 수 있다.
cpu.shares가 100으로 동일하게 설정된 두 cgroup의 작업은 동일한 비율의 cpu 시간을 사용할 수 있지만
cpu.shares가 200으로 설정된 cgroup의 작업은 cpu.shares가 설정된 cgroup의 작업보다 2배의 cpu 시간이 허용된다.

cpu.cfs_period_us
: cpu 리소스에 대한 cgroup의 접근이 이루어지는 주기를 나타낸다. 단위는 마이크로초이며 상한은 1초, 하한은 1000마이크로초이다.

cpu.cfs_qouta_us
: cgroup의 작업이 한 주기(cpu.cfs_period_us) 동안 실행될 수 있는 총 시간을 나타낸다. 역시 마이크로초 단위이며 cgroup에 할당된 시간을 사용하는 즉시 나머지 시간 동안은 접근이 제한되고
다음 주기가 올 때까지 실행이 허용되지 않는다. cgorup의 작업이 1초 중 0.2초 동안 단일 cpu에 접근할 수 있어야 하는 경우 cpu.cfs_quota_us를 200,000으로 설정하고
cpu.cfs_period_us를 1,000,000으로 설정하면 된다. 또한 cgroup 내의 프로세스가 2개의 cpu를 사용할 수 있게 허용하려면 cpu.cfs_quota_us를 200,000으로,
cpu.cfs_period_us를 100,000으로 설정하면 된다. 이 값이 -1(기본값)일 경우는 제한이 없다는 뜻이다.

(https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/resource_management_guide/sec-cpu)

cgrpus v2의 경우는 [Set](https://github.com/opencontainers/runc/blob/main/libcontainer/cgroups/fs2/cpu.go#L17)
에 의해 다음과 같이 설정된다.

- cpuShares -> [cpuWeight](https://github.com/kubernetes/kubernetes/blob/master/pkg/kubelet/cm/cgroup_manager_linux.go#L363) -> cpu.weight
- cpuQuota cpuPeriod -> cpu.max

cpu.weight
: 가중치 기반 cpu 시간 분포 모델 처리를 위한 값이다. 기본값은 100이며,
cpu 시간을 하위 cgroup들에게 분배할 때 모든 cpu.weight 값을 합산하여 가중치 비례에 따른 상대적인 cpu 시간을 얻을 수 있도록 한다.
즉, 모든 cpu.weight 파일의 값이 동일하면 모든 하위 cgroup들은 동일한 cpu 시간을 사용할 수 있게 된다.
이 값은 서로 다른 경우에만 중요하다.

cpu.max
: 최대 대역폭 제한이며, $MAX $PERIOD 형식으로 설정된다.
각 cgroup이 period 기간 내에서 최대 max까지 cpu를 사용할 수 있음을 나타낸다.
$MAX에 max를 기입하면 제한이 없음을 나타낸다.

(https://docs.kernel.org/admin-guide/cgroup-v2.html)

memory쪽은 일단 이정도만 보자.

```shell
➜  4b96c3c56e7a5b1596ff64c58cfa7fdad52de3017918071fcc3229091b4f0c7e grep "" memory.*
memory.current:237568
...
memory.high:max
memory.low:0
memory.max:max
memory.min:0
...
```

memory.high
: 메모리 사용량에 대한 throttle(soft) 제한이다.
cgroup의 메모리 사용량이 여기에 지정된 상한선을 넘어가면 cgroup의 프로세스는 스로틀링이 걸리고 상당한 메모리 회수에 대한 압박을 받게 된다.
여기에서 프로세스에 작용되는 스로틀링은 상한선을 넘어가는 순간 메모리를 스왑해 디스크로 옮기고 최대한 상한선을 유지하도록 한다.
기본값은 max이며 제한이 없음을 의미한다.

memory.low
: 이 값만큼의 메모리는 회수되지 않는다는 soft 보증을 위한 설정이다.

memory.max
: 메모리 사용량에 대한 hard 제한이다.
cgroup의 메모리 사용량이 이 제한에 도달하고 줄일 수 없게 되면 OOM Killer가 호출되어 매커니즘에 따라 프로세스를 죽이게 된다.

memory.min
: cgroup이 확보해야하는 최소한의 메모리를 나타낸다.
해당 값을 확보할 수 없는 경우에 OOM Killer가 호출된다.

(https://facebookmicrosites.github.io/cgroup2/docs/memory-controller.html)

oom_score도 보자.
QoS가 BestEffort이므로 oom_score_adj는 1000으로 설정되어 있다.

```shell
➜  ~ grep "" /proc/616691/oom_score*
/proc/616691/oom_score:1332
/proc/616691/oom_score_adj:1000
```

## Burstable

## Guaranteed

---

[^1]: https://man7.org/linux/man-pages/man7/cgroups.7.html
