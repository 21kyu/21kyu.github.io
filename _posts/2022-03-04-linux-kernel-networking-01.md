---
title: Socket Buffer
author: wq
name: Wongyu Lee
link: https://github.com/kyu21
date: 2022-03-04 20:00:00 +0900
categories: [linux]
tags: [kernel, network]
render_with_liquid: false
---

커널 내부 구조를 이해하면 eBPF를 이해하는데 도움이 될 것이다.
여기서 핵심은 패킷이 `struct sk_buff(socket buffer)` 구조체를 통해 커널 네트워크 스택을 통과한다는 것이다.
(커널 영역) 소켓의 경우 `struct sock` 구조체에 정의되어 있는데, 이 구조체는 `tcp_sock`과 같은 소켓 프로토콜의 앞부분에 내장되어 있다.
네트워크 프로토콜은 `tcp_prot`, `udp_prot` 등의 `struct proto` 구조체를 사용해서 소켓에 연결된다.

우선 `sk_buff`를 조사해보자.

_[리눅스 커널 네트워킹](http://www.kyobobook.co.kr/product/detailViewKor.laf?ejkGb=KOR&mallGb=KOR&barcode=9791158390471)
의 부록 A. 리눅스 API 중 sk_buff 구조체를 정리해두고자 한다._

### socket buffer?

패킷의 이동을 더 잘 이해하려면 리눅스 커널에서 패킷이 어떻게 표현되는지 알면 알수록 좋다고 한다.
리눅스 커널에서 소켓 버퍼(socket buffer)는 `sk_buff` 구조체로 표현되며 유입/유출 패킷을 나타낸다.

패킷은 로컬 장비에서 로컬 소켓으로 생성될 수 있으며(_로컬 호스트에서 생성된 패킷은 4계층 소켓:TCP 또는 UDP 소켓을 통해 만들어진다._),
(_소켓 API를 통해_) 사용자 공간 애플리케이션에서 생성된다. 소켓에는 데이터그램(datagram) 소켓과 스트림(stream) 소켓이라는 두 가지 주된 유형이 있다.
로컬에서 생성된 패킷은 네트워크 계층인 L3로 전달(_여기서 단편화가 발생하기도 한다._)된 후 전송을 위해 네트워크 장치 드라이버(L2)로 전달된다.
패킷은 외부 또는 같은 장비 내의 다른 소켓으로 보내질 수 있다.

또한 패킷은 커널 소켓에 의해 생성될 수도 있다.
네트워크 장치(L2)로부터 물리적 프레임을 수신하고 이를 `sk_buff`에 연결(_netdev_alloc_skb() 함수를 호출하여 sk_buff 구조체를 할당한다_)한 후 네트워크 계층(L3)에 전달할 수 있다.
패킷 목적지가 로컬 장비이면 전송 계층(L4)으로 계속 이동할 것이고, 패킷 목적지가 로컬 장비가 아니라면 라우팅 테이블 규칙에 따라 포워딩될 것이다(로컬 장비가 포워딩을 지원할 경우).
패킷은 이유가 어떻든 간에 손상되면 폐기된다.

> 참고: SKB나 sk_buff라고도 부르는 소켓 버퍼(socker buffer) 구조체는 전송 또는 수신된 모든 패킷에 대해 커널이 생성해서 사용하는 구조체이다. BPF 프로그램은 이 SKB를 읽어서 패킷 전달 여부를 결정하거나, 현재 소통량에 관한 통계치와 측정치를 BPF 맵에 추가한다. 또한 eBPF는 SKB의 수정도 가능하다. 최종 패킷의 대상 주소를 변경함으로써 패킷을 다른 곳으로 재지정할 수 있으며, 심지어 근본적인 구조고 바꿀 수 있다. 이런 능력을 활용하면 IPv6 전용시스템이 받은 모든 패킷을 IPv4로 변환하는 것도 가능하다. (Linux Observability with BPF)

### sk_buff struct

sk_buff 구조체를 살펴보자.

```c
struct sk_buff {
  union {
    struct {
      /* These two members must be first to match sk_buff_head. */
      struct sk_buff    *next;
      struct sk_buff    *prev;

      union {
        // dev는 SKB와 연관된 네트워크 인터페이스 장치를 나타내는 net_device 객체이다.
        // 간혹 그러한 네트워크 장치에 대한 NIC(network interface card)라는 용여를 접하게 될 것이다.
        // NIC는 패킷이 도착한 네트워크 장치나 패킷을 보낼 네트워크 장치가 될 수 있다.
        // 다음 포스팅에서 좀 더 자세히 정리해두자.
        struct net_device  *dev;
        /* Some protocols might use this space to store information,
         * while device pointer would be NULL.
         * UDP receive path is one user.
         */
        unsigned long    dev_scratch;
      };
    };
    struct rb_node    rbnode; /* used in netem, ip4 defrag, and tcp stack */
    struct list_head  list;
    struct llist_node  ll_node;
  };

  union {
    // SKB(로컬 생성 트래픽과 로컬 호스트를 목적지로 하는 트래픽에 대한)를 소유한 소켓이다. 포워딩될 패킷은 sk가 NULL이다.
    // 보통 소켓에 관해 이야기할 때는 사용자 공간에서 socket() 시스템 콜을 호출해 생성된 소켓을 기준으로 한다.
    // sock_create_kern() 함수를 호출해 생성되는 커널 소켓에 대해서도 알아둘 필요가 있다.
    struct sock    *sk;
    int      ip_defrag_offset;
  };

  union {
    // 패킷의 도착 timestamp. SKB에서 기본 timestamp로 제공돼 저장된다.
    ktime_t    tstamp;
    u64    skb_mstamp_ns; /* earliest departure time */
  };
  /*
   * This is the control buffer. It is free to use for every
   * layer. Please put your private variables there. If you
   * want to keep them across layers you have to do a skb_clone()
   * first. This is owned by whoever has the skb queued ATM.
   */
  // 제어 버퍼로서 다른 계층에서 자유롭게 사용될 수 있다.
  // 이것은 비공개 정보를 저장하는데 사용되는 불투명 영역으로 TCP 프로토콜에서는 TCP 제어 버퍼를 다음과 같이 사용한다.
  // #define TCP_SKB_CB(__skb) ((struct tcp_skb_cb *)&((__skb)->cb[0])) @include/net/tcp.h
  char      cb[48] __aligned(8);

  union {
    struct {
      // 목적지 항목(dst_entry) 주소. dst_entry 구조체는 특정 목적지에 대한 라우팅 항목을 나타낸다.
      // 각 송수신 패킷의 경우 라우팅 테이블에서 탐색을 수행한다.
      // 간혹 이 탐색을 FIB(Forwarding Information Base) 탐색이라 하며 이 탐색의 결과에 따라 이 패킷을 어떻게 처리할 지 결정된다.
      // 포워딩돼야 하는지, 포워딩돼야 한다면 어떤 인터페이스를 사용해 전송될지, 또는 던져져야 하는지, ICMP 오류 메시지를 보내야 하는지
      unsigned long  _skb_refdst;
      // kfree_skb() 함수를 호출해 SKB를 해제할 때 호출되는 콜백
      void    (*destructor)(struct sk_buff *skb);
    };
    struct list_head  tcp_tsorted_anchor;
#ifdef CONFIG_NET_SOCK_MSG
    unsigned long    _sk_redir;
#endif
  };

#if defined(CONFIG_NF_CONNTRACK) || defined(CONFIG_NF_CONNTRACK_MODULE)
  unsigned long     _nfct;
#endif
  // 패킷 바이트의 전체 개수
  unsigned int    len,
  // 데이터 길이. 이 필드는 패킷이 비선형 데이터(페이지된 데이터)를 가질 때만 사용된다.
        data_len;
  // MAC(2계층) 헤더의 길이
  __u16      mac_len,
        hdr_len;

  /* Following fields are _not_ copied in __copy_skb_header()
   * Note that queue_mapping is here mostly to fill a hole.
   */
  __u16      queue_mapping;

/* if you move cloned around you also must adapt those constants */
#ifdef __BIG_ENDIAN_BITFIELD
#define CLONED_MASK  (1 << 7)
#else
#define CLONED_MASK  1
#endif
#define CLONED_OFFSET    offsetof(struct sk_buff, __cloned_offset)

  /* private: */
  __u8      __cloned_offset[0];
  /* public: */
  // 패킷이 __skb_cloen() 함수로 복제되면 이 필드는 복제 패킷과 주 패킷 모두 1로 설정된다.
  // SKB 복제는 sk_buff struct의 사본을 생성하는 것을 의미한다. 데이터 영역은 복제본과 주 SKB 사이에 공유된다.
  __u8      cloned:1,
        nohdr:1,
        fclone:2,
  // 이 패킷은 이미 확인되어 이 패킷에 대한 통계 작업(?)이 이뤄졌다. 따라서 다시 통계 작업을 수행하지 않는다.
        peeked:1,
        head_frag:1,
        pfmemalloc:1,
        pp_recycle:1; /* page_pool recycle indicator */
#ifdef CONFIG_SKB_EXTENSIONS
  __u8      active_extensions;
#endif

  /* Fields enclosed in headers group are copied
   * using a single memcpy() in __copy_skb_header()
   */
  struct_group(headers,

  /* private: */
  __u8      __pkt_type_offset[0];
  /* public: */
  // 이더넷의 경우 패킷 타입은 이더넷 헤더의 목적지 MAC 주소에 좌우되며, eth_type_trans() 함수로 결정된다.
  // 브로드캐스트에 대한 PACKET_BROADCAST
  // 멀티캐스트에 대한 PACKET_MULTICAST
  // 목적지 MAC 주고가 매개변수로 전달된 장치의 MAC 주소면 PACKET_HOST
  // 이 조건이 일치하지 않으면 PACKET_OTHERHOST
  __u8      pkt_type:3; /* see PKT_TYPE_MAX */
  __u8      ignore_df:1;
  // 넷필터 패킷 추적 플래그. 이 플래그는 패킷 흐름 추적 넷필터 모듈인 xt_TRACE 모듈로 설정되며, 추적할 패킷을 표시하는데 사용된다.
  __u8      nf_trace:1;
  __u8      ip_summed:2;
  __u8      ooo_okay:1;

  __u8      l4_hash:1;
  __u8      sw_hash:1;
  __u8      wifi_acked_valid:1;
  __u8      wifi_acked:1;
  __u8      no_fcs:1;
  /* Indicates the inner headers are valid in the skbuff. */
  // 캡슐화 필드는 SKB가 캡슐화에 사용됨을 의미한다.
  // 예를 들면 VXLAN 드라이버에서 사용된다. VXLAN은 UDP 커널 소켓을 통해 2계층 이더넷 패킷을 전송하는 표준 프로토콜이다.
  // 이 프로토콜은 터널을 차단하고 TCP나 UDP 통신만 허용하는 방화벽이 있을 경우 솔루션으로 사용될 수 있다.
  // VXLAN 드라이버는 UDP 캡슐화를 사용하고 SKB 캡슐화를 vxlan_init_net() 함수에서 1로 설정한다.
  __u8      encapsulation:1;
  __u8      encap_hdr_csum:1;
  __u8      csum_valid:1;

  /* private: */
  __u8      __pkt_vlan_present_offset[0];
  /* public: */
  __u8      vlan_present:1;  /* See PKT_VLAN_PRESENT_BIT */
  __u8      csum_complete_sw:1;
  __u8      csum_level:2;
  __u8      csum_not_inet:1;
  __u8      dst_pending_confirm:1;
#ifdef CONFIG_IPV6_NDISC_NODETYPE
  __u8      ndisc_nodetype:2;
#endif

  // 이 플래그는 SKB가 ipvs(IP 가상 서버)를 소유했는지를 나타낸다.
  // IPVS는 커널 기반 전송 계층 로드 밸런싱 솔루션이다.
  __u8      ipvs_property:1;
  __u8      inner_protocol_type:1;
  __u8      remcsum_offload:1;
#ifdef CONFIG_NET_SWITCHDEV
  __u8      offload_fwd_mark:1;
  __u8      offload_l3_fwd_mark:1;
#endif
#ifdef CONFIG_NET_CLS_ACT
  __u8      tc_skip_classify:1;
  __u8      tc_at_ingress:1;
#endif
  __u8      redirected:1;
#ifdef CONFIG_NET_REDIRECT
  __u8      from_ingress:1;
#endif
#ifdef CONFIG_NETFILTER_SKIP_EGRESS
  __u8      nf_skip_egress:1;
#endif
#ifdef CONFIG_TLS_DEVICE
  __u8      decrypted:1;
#endif
  __u8      slow_gro:1;

#ifdef CONFIG_NET_SCHED
  __u16      tc_index;  /* traffic control index */
#endif

  union {
    // 체크섬
    __wsum    csum;
    struct {
      __u16  csum_start;
      __u16  csum_offset;
    };
  };
  // 패킷의 큐 우선순위. Tx 경로에서 SKB의 우선순위는 소켓 우선순위(소켓의 sk_priority 필드)에 따라 설정된다.
  // 소켓 우선순위는 SO_PRIORITY 소켓 옵션으로 setsocket() 시스템 콜을 호출해 설정될 수 있다.
  // net_prio cgroup 커널 모듈을 사용하면 SKB에 대한 우선순위를 설정할 규칙을 정의할 수 있다.
  // 포워딩 패킷에 대한 우선순위는 IP 허데에서 TOS(서비스 타입) 필드에 따라 설정된다.
  // 16개의 요소로 구성된 ip_tos2prio라고 하는 테이블이 있는데 TOS에서 우선순위로 변환하는 것은 rt_tos2priority() 함수로 이뤄지며, 이때 IP 헤더의 TOS 필드에 따른다.
  __u32      priority;
  int      skb_iif;
  __u32      hash;
  // VLAN 프로토콜이 사용됨. 보통 이는 802.1q 프로토콜이다.
  __be16      vlan_proto;
  // VLAN 태그 제어 정보(2바이트). ID와 우선순위로 구성된다. (SPT의 Bridge id, priority는 아니겠지..?)
  __u16      vlan_tci;
#if defined(CONFIG_NET_RX_BUSY_POLL) || defined(CONFIG_XPS)
  union {
    unsigned int  napi_id;
    unsigned int  sender_cpu;
  };
#endif
#ifdef CONFIG_NETWORK_SECMARK
  // 보안 표시 필드. secmark 필드는 iptables SECMARK 대상으로 설정되며, 이는 패킷을 유효한 보안 컨텍스트로 라벨링한다.
  // 예를 들면,
  // iptables -t mangle -A INPUT -p tcp --dport 80 -j SECMARK --selctx
  // system_u:object_r:httpd_packet_t:s0
  // iptables -t mangle -A OUTPUT -p tcp --sport 80 -j SECMARK --selctx
  // system_u:object_r:httpd_packet_t:s0
  // 위 규칙에서 80번 포트에서 송수신되는 패킷을 httpd_packet_t로 고정적으로 라벨링한다.
  __u32    secmark;
#endif

  union {
    // 이 필드는 SKB를 표시해 식별할 수 있게 한다.
    // 예를 들면, SKB의 mark 필드를 설정할 수 있는데, 조각(mangle) 테이블의 iptables PREROUTING 규칙에서 iptalbes MARK 대상으로 설정할 수 있다.
    // iptables -A PREROUTING -t mangle -i eth1 -j MARK --set-mark 0x1234
    // 이 규칙은 탐색을 수행하기 ㅈㅓsdp eth1의 수신 트래픽을 대상으로 모든 SKB mark 필드에 0x1234 값을 할당할 것이다.
    __u32    mark;
    __u32    reserved_tailroom;
  };

  union {
    __be16    inner_protocol;
    __u8    inner_ipproto;
  };

  // 전송 계층(L4) 헤더
  __u16      inner_transport_header;
  // 네트워크 계층(L3) 헤더
  __u16      inner_network_header;
  // 링크 계층(L2) 헤더
  __u16      inner_mac_header;

  __be16      protocol;
  __u16      transport_header;
  __u16      network_header;
  __u16      mac_header;

#ifdef CONFIG_KCOV
  u64      kcov_handle;
#endif

  ); /* end headers group */

  /* These elements must be at the end, see alloc_skb() for details.  */
  // 데이터의 꼬리(tail)
  sk_buff_data_t    tail;
  // 버퍼가 끝나는 부분. tail은 end를 초과할 수 없다.
  sk_buff_data_t    end;
  // 버퍼의 머리(head)
  unsigned char    *head,
  // 데이터의 머리(head). 데이터 영역은 sk_buff 할당에서 분리되어 할당된다.
        *data;
  // SKB에 대해 할당된 전체 메모리(SKB 구조체 자체와 할당된 데이터 영역 크기를 포함)
  unsigned int    truesize;
  refcount_t    users;

#ifdef CONFIG_SKB_EXTENSIONS
  /* only useable after checking ->active_extensions != 0 */
  struct skb_ext    *extensions;
#endif
};
```
{: .nolineno}

#### Basic functions for sk_buff

SKB의 headroom과 tailroom은 아래와 같다.

<img alt="skb" src="/images/skb.png" width="300"/>

참고: [Basic functions for sk_buff{}](http://www.skbuff.net/skbbasic.html)

void skb_reserve(struct sk_buff *skb, int len)
: tail을 감소시켜 빈 skb의 headroom을 증가시킨다.
(headroom을 조정한다.)

void *skb_push(struct sk_buff *skb, unsigned int len)
: skb의 데이터 포인터를 감소시키고 skb의 길이를 len만큼 증가시킨다.
(headroom을 줄여서 데이터를 추가한다.)

void *skb_pull(struct sk_buff *skb, unsigned int len)
: skb의 데이터 포인터를 증가시키고 len만큼 skb의 길이를 감소시킨다.
(headroom을 늘려서 데이터를 제거한다.)

void *skb_put(struct sk_buff *skb, unsigned int len)
: 버퍼에 데이터를 추가한다. 이 함수는 skb의 버퍼에 len 바이트를 추가해서 skb의 길이를 len만큼 증가시킨다.
(tailroom을 줄이고 데이터를 추가한다.)

void skb_trim(struct sk_buff *skb, unsigned int len)
: 버퍼의 데이터를 삭제한다.
(tailroom을 늘려 데이터를 제거한다.)
