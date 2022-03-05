---
title: Network Device
author: wq
name: Wongyu Lee
link: https://github.com/kyu21
date: 2022-03-06 02:00:00 +0900
categories: [linux]
tags: [kernel, network]
render_with_liquid: false
---

리눅스 커널 네트워킹 스택에서 처리하는 3계층 중 가장 낮은 계층인 L2는 데이터 링크 계층이다.
네트워크 장치 드라이버는 이 계층에 속하며 이와 관련된 기본적인 사항에 익숙해지면 네트워크 스택을 이해하는데 큰 도움이 될 것이다.

## Network Interface

네트워크 인터페이스는 네트워크 연결을 위한 운영체제의 끝점이다.
인터페이스는 시스템 관리자가 설정하고 관리하는 추상적인 요소다.

![network interface](/images/network-interface.png){: width="500" }
_network interface_

네트워크 인터페이스는 설정 정보에 있는 물리적인 네트워크 포트와 매핑된다.
포트는 네트워크에 연결되며, 보통 수신과 송신 채널을 따로 가지고 있다.

## Network Controller

NIC(Network Interface Card)는 시스템에 네트워크 포트를 하나 이상 제공하며, 네트워크 컨트롤러를 수용한다.
컨트롤러는 포트와 시스템 I/O 트랜스포트 사이에 패킷을 전송하는 마이크로 프로세서다.

![network controller](/images/network-controller.png){: width="500" }
_network controller_

컨트롤러는 별도의 카드로 돼 있거나 시스템 보드에 내장돼 있다.

## Network Device Drivers

네트워크 장치 드라이버(Network Device Drivers)에는 보통 커널 메모리와 NIC 사이에 데이터를 보내고 받기 위한 추가 버퍼(보통 링 버퍼)가 있다.

### Network Device의 NAPI(New API)

오래된 네트워크 장치 드라이버는 인터럽트 구동 모드(interrupt-driven mode)로 동작하며 이는 패킷을 수신할 때마다 인터럽트가 발생한다는 의미다.
이 방식은 트래픽 부하가 심할 때 성능 측면에서 비효율적인 것으로 증명됐다.
그리하며 New API(NAPI)라고 하는 새로운 소프트웨어 기법이 개발됐고 현재 대부분의 리눅스 네트워크 장치 드라이버에서 지원한다.

![linux low level network stack](/images/linux-low-level-network-stack.png){: width="500" }
_linux low level network stack_

NAPI를 이용하면 부하가 높은 상태에서 네트워크 장치 드라이버가 인터럽트 구동 방식이 아닌 폴링 방식(polling mode)으로 동작한다.
각 패킷은 드라이버에 버퍼링되고, 커널이 이따금 패킷을 가져오기 위해 드라이버를 대상으로 폴링한다.
NAPI를 사용하면 부하가 높은 상황일 때 성능이 향상된다.

### Network Device Driver의 주요 작업

네트워크 장치 드라이버의 주요 작업은 다음과 같다.

* 로컬 호스트를 목적지로 하는 패킷을 수신하고, 네트워크 계층(L3)을 거쳐 전송 계층(L4)로 전달
* 로컬 호스트에서 생성되어 외부로 나가는 패킷을 전송하거나, 로컬 호스트에서 수신된 패킷을 포워딩

### Busy Polling on Socket

소켓 큐가 고갈될 때 네트워킹 스택이 동작하는 전통적인 방법은 드라이버가 소켓 큐에 데이터를 넣을 수 있을 때까지 기다리며 잠들거나, 비차단 동작일 경우 반환하는 것이다.
이는 인터럽트와 컨텍스트 전환으로 인해 추가적인 지연을 발생시킨다.
지연 시간을 최대한 낮춰야 하지만 높은 CPU 사용률은 기꺼이 감수할 수 있는 소켓 애플리케이션에 대해 리눅스는 바쁜 폴링(busy polling) 기능을 추가했다.
바쁜 폴링은 애플리케이션으로 데이터를 옮기는 방향으로 더욱 적극적인 접근법을 취한다.
애플리케이션이 데이터를 더 요청하고 소켓 큐에 아무것도 없다면 네트워킹 스택은 장치 드라이버를 호출한다.
드라이버는 새로 도착한 데이터를 검사하고 네트워크 계층(L3)을 통해 소켓에 데이터를 밀어 넣는다.

바쁜 폴링은 하드웨어에 가까운 지연시간을 제공한다.
대량의 소켓에 동시에 사용될 수 있지만 최상의 결과를 낼 수는 없는데, 일부 소켓에 대한 바쁜 폴링이 동일한 CPU 코어를 사용하는 다른 소켓을 느리게 하기 때문이다.

* 전통적인 수신 경로 흐름

1. 애플리케이션이 수신 검사를 한다.
2. 소켓 큐에 데이터가 없어서 즉시 받을 수 없으므로 차단된다.
3. 패킷이 수신돼 NIC에 도착하고 장치 드라이버로 전달된다.
4. 장치 드라이버는 프로토콜 계층에 패킷을 전달한다.
5. 프로토콜/소켓이 애플리케이션을 깨운다. - Bypass context switch and interrupt.
6. 애플리케이션은 소켓을 통해 데이터를 수신한다. 반복

* 바쁜 폴링이 적용된 소켓의 수신 흐름

1. 애플리케이션이 수신 검사를 한다.
2. 네트워킹 스택은 보류 패킷에 대해 장치 드라이버를 검사한다. (폴링 시작)
3. 그 사이 NIC에 패킷이 수신되고 장치 드라이버로 전달된다.
4. 장치 드라이버가 프로토콜 계층에 패킷을 전달한다.
5. 애플리케이션은 소켓을 통해 데이터를 수신한다. 반복

## net_device 구조체

net_device 구조체는 네트워크 장치를 나타내며, 이것은 이더넷 장치 같은 물리 장치가 될 수 있거나 브리지 장치나 VLAN 장치 같은 소프트웨어 장치가 될 수 있다.
장치 매개변수는 패킷을 분할해야 하는지 결정한다. net_device는 매우 큰 구조체이고, 다음과 같은 장치 매개변수로 구성돼 있다.

* IRQ 번호
* MTU
* MAC 주소
* 장치의 이름(e.g., eth0 또는 eth1)
* 플래그 값(e.g., 활성 또는 비활성)
* 연관된 멀티캐스트 주소 목록
* promiscuity(무차별) 카운터
* 장치가 지원하는 기능(e.g., GSO 또는 GRO 오프로딩)
* 네트워크 장치 콜백 객체(net_device_ops 객체): 장치를 열거나 종료, 전송 시작, 네트워크 장치의 MTU 변경 등을 위한 함수 포인터로 구성돼 있다.
* ethtool 콜백 객체: 명령줄 ethtool 유틸리티를 실행해 장치에 관한 정보를 수집할 수 있게 지원한다.
* 멀티 큐를 지원하는 장치일 경우 Tx, Rx 큐의 개수
* 패킷의 마지막 송신/수신 타임스탬프

*net_device 구조체의 일부*

```c
struct net_device {
  unsinged int irq; /* 장치 IRQ 번호 */
  ...
  const strict net_device_ops *netdev_ops;
  ...
  unsigned int mtu;
  unsigned int promiscuity;
  unsigned char *dev_adr;
  ...
```
{: .nolineno}
