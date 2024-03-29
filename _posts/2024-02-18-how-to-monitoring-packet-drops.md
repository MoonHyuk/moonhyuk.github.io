---
title: 리눅스 네트워크 모니터링 1편 - 호스트에서 드랍된 패킷 모니터링
tags:
  - Linux Networking
  - TCP/IP
  - node_exporter
  - Ftrace
---

## 개요
서비스를 운영하다보면 네트워크 문제로 인해 어플리케이션 응답이 지연되는 것을 종종 경험합니다. 네트워크 문제는 어플리케이션에서는 인지하기가 어려워 문제 원인을 찾는 데에 시간이 오래 걸리기도 합니다. 다행히 리눅스의 여러 기능들을 통해 네트워크 문제를 직간접적으로 모니터링할 수 있습니다. 

네트워크 문제는 호스트 내에서도 발생할 수도 있고 호스트 외부 네트워크에서 발생할 수도 있습니다. 이 글에서는 호스트 내부에서 패킷이 드랍되는 문제를 모니터링하는 방법을 소개드리겠습니다. 

리눅스의 네트워킹 스택은 매우 복잡하여 그만큼 다양한 구간에서 패킷 유실이 발생할 수 있습니다. 최근에는 DPDK, XDP 등으로 커널 네트워킹 스택을 우회하거나 커널 안에 원하는 기능을 추가할 수 있게 되며 다른 모니터링 방법도 필요해졌습니다. 이 글에서는 전통적인 리눅스 네트워킹 스택의 모니터링만 다룹니다.

## 메트릭
리눅스에서는 네트워크 계층별로 다양한 통계 정보를 제공합니다. 계층별 통계를 확인하는 명령어가 달라 어렵기도 합니다. 명령어들을 외워 사용하는 것 보다는 node_exporter 등을 통해 다양한 메트릭을 자동으로 수집하고 모니터링하는 것이 좋습니다.

계층별로 제공하는 패킷 유실 관련 메트릭들과, 이를 확인하는 명령어, 그리고 node_exporter 1.7 버전에서 이를 수집하게 하려면 어떤 옵션을 주어야 하는지 알아보겠습니다.

### 링크 계층
우선 링크 계층 관련 표준 통계는 `ip -s -s link show dev ${name}` 명령어로 조회할 수 있습니다.
```shell
$ ip -s -s link show dev eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 02:f3:0f:08:44:fa brd ff:ff:ff:ff:ff:ff
    RX:  bytes packets errors dropped  missed   mcast
    1082991757 3293770      0       0       0       0
    RX errors:  length    crc   frame    fifo overrun
                     0      0       0       0       0
    TX:  bytes packets errors dropped carrier collsns
    1359059421 2014325      0       0       0       0
    TX errors: aborted   fifo  window heartbt transns
                     0      0       0       0       1
```
각 필드에 대한 설명은 리눅스 문서에 설명되어 있습니다.[^1]
프레임 유실과 관련된 통계들은 `rx_errors`, `tx_errors`, `rx_dropped`, `tx_dropped` 등이 있고, 더 자세한 오류 통계는 `rx_length_errors`, `rx_fifo_errors`, `rx_missed_errors`, `tx_aborted_errors`, `tx_fifo_errors` 등의 필드로 확인할 수 있습니다.

node_exporter에서는 `--collector.netdev` 옵션과 `--collector.netdev.enable-detailed-metrics` 옵션을 통해 관련 메트릭을 수집할 수 있습니다.

`--collector.netdev` 옵션은 기본으로 켜져 있고, 아래 메트릭들을 수집합니다.
```text
node_network_receive_drop_total
node_network_receive_errs_total
node_network_receive_fifo_total
node_network_receive_frame_total
node_network_transmit_carrier_total
node_network_transmit_colls_total
node_network_transmit_drop_total
node_network_transmit_errs_total
node_network_transmit_fifo_total
```

`--collector.netdev.enable-detailed-metrics`옵션은 더 자세한 메트릭을 수집합니다. 이 옵션은 기본으로 꺼져있습니다. 수집을 원한다면 명시적으로 켜주어야 합니다. 이 옵션을 켜면 아래 메트릭들이 추가로 수집됩니다.
```text
node_network_receive_crc_errors_total
node_network_receive_dropped_total
node_network_receive_errors_total
node_network_receive_fifo_errors_total
node_network_receive_frame_errors_total
node_network_receive_length_errors_total
node_network_receive_missed_errors_total
node_network_receive_over_errors_total
node_network_transmit_aborted_errors_total
node_network_transmit_carrier_errors_total
node_network_transmit_dropped_total
node_network_transmit_errors_total
node_network_transmit_fifo_errors_total
node_network_transmit_heartbeat_errors_total
node_network_transmit_window_errors_total
```

리눅스의 표준 링크 통계 이외에도, 네트워크 드라이버가 제공하는 통계가 있습니다. `ethtool -S ${디바이스 이름}` 명령어를 통해 이를 확인할 수 있습니다.
제공되는 정보는 네트워크 드라이버에 따라 다릅니다.

node_exporter를 통해 네트워크 드라이버가 제공하는 통계를 수집하려면 `--collector.ethtool` 옵션을 켜주어야 합니다. 이 옵션은 기본적으로 꺼져 있습니다.

### CPU 대기열
멀티코어 시스템에서 네트워크 처리를 분산시키는 기술들[^2] 중 Receive Packet Steering(RPS)는 커널에서 CPU 별로 대기열을 만듭니다. 네트워크 드라이버가 패킷을 받아 `netif_rx()` 혹은 `netif_receive_skb()` 함수를 호출하면, `get_rps_cpu()` 함수가 실행되어 패킷을 전달할 대기열이 결정됩니다. 그러나 이때 대기열의 길이가 `sysctl net.core.netdev_max_backlog` 값을 넘으면 (실제로는 좀 더 복잡한 계산을 합니다.) 패킷이 유실됩니다.

이때 유실된 패킷의 수는 `/proc/net/softnet_stat` 파일의 두 번째 열 값을 통해 확인할 수 있습니다.
```shell
$ cat /proc/net/softnet_stat
03cd3c87 00000000 0000026c 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
139c9461 00000000 0000cd8d 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
```

node_exporter에서는 `--collector.softnet` 옵션을 키면 이 통계를 수집할 수 있습니다. 기본적으로 켜져 있습니다.
```text
node_softnet_dropped_total
```

### File descriptor
TCP 커넥션이 맺어지면 소켓이 생성되고, 소켓이 생성되면 file descriptor를 만들게 됩니다. 이때 리눅스에서 최대로 만들 수 있는 File descriptor 수가 제한되어 있다는 것을 주의해야 합니다. 시스템의 전체 제한은 `fs.file-max` 커널 파라미터 값을 따릅니다.
```bash
$ sysctl fs.file-max
fs.file-max = 655360
```
시스템에서 현재 사용 중인 총 file descriptor 수는 `/proc/sys/fs/file-nr` 파일의 첫 번째 열을 통해 확인할 수 있습니다.
```bash
$ cat /proc/sys/fs/file-nr
2560	0	655360
```
File descriptor의 시스템 전체 제한 외에도 프로세스 별 제한도 있습니다. `ulimit -n` 명령어를 통해 확인할 수 있습니다.
```bash
$ ulimit -n
65536
```

node_exporter에서는 `--collector.filefd` 옵션을 통해 file descriptor 관련 통계를 수집할 수 있습니다. 이 옵션은 기본으로 켜져 있습니다.
하지만 시스템 전체의 file descriptor 제한과 현재 사용량만을 알 수 있고, 프로세스 별 제한을 수집하지 않는 걸로 보입니다.

```text
node_filefd_allocated
node_filefd_maximum
```

### Netfilter Conntrack
Netfilter Conntrack(Connection Tracking) 리눅스 stateful 방화벽 시스템에서 커넥션 상태 테이블을 관리하는 역할을 수행합니다. Conntrack 테이블 크기에 제한이 있기 때문에 커넥션이 많은 상황에서는 패킷 유실이 발생할 수 있습니다.

Conntrack 테이블의 최대 크기는 `/proc/sys/net/netfilter/nf_conntrack_max`를 통해 확인 가능합니다.
```bash
$ cat /proc/sys/net/netfilter/nf_conntrack_max
262144
```

현재 conntrack 테이블 크기는 `/proc/sys/net/netfilter/nf_conntrack_count`를 통해 확인 가능합니다.
```bash
$ cat /proc/sys/net/netfilter/nf_conntrack_count
16392
```

node_exporter에서는 `--collector.conntrack` 옵션으로 이 통계를 수집할 수 있습니다. 이 옵션은 기본으로 켜져 있습니다.
```text
node_nf_conntrack_entries
node_nf_conntrack_entries_limit
```

### Traffic Control
Traffic Control(TC)는 iproute2 패키지에 포함되어 있는 유틸리티로, 네트워크 전송 속도를 제어하거나, 대기열 스케쥴링, 패킷 드랍 등을 수행합니다.

TC 정책에 따라 패킷이 드랍될 수도 있고, 또는 대기열이 꽉 차 패킷이 유실될 수도 있습니다. 이 단계에서 버려진 패킷 수는 `tc -s qdisc show` 명령어로 확인합니다.
```shell
$ tc -s qdisc show
qdisc htb 1: dev eth0 root refcnt 5 r2q 10 default 0x1 direct_packets_stat 0 direct_qlen 50000
 Sent 625485912703 bytes 1381641641 pkt (dropped 0, overlimits 94242668 requeues 3106)
 backlog 0b 0p requeues 3106
```

node_exporter에서는 `–-collector.qdisc` 옵션을 통해 이 통계를 수집할 수 있습니다. 이 옵션은 기본적으로 꺼져있습니다.
```text
node_qdisc_drops_total
```

### 네트워크 및 전송 계층
네트워크 및 전송 계층에서는 매우 많은 통계 정보를 제공합니다. `nstat` 명령어를 통해 확인 가능합니다.

리눅스 커널 6.2 기준으로 360개의 통계를 제공하고 있습니다.
```bash
$ nstat -az | wc -l
361

$ nstat -az
#kernel
IpInReceives                    5198909            0.0
IpInHdrErrors                   0                  0.0
IpInAddrErrors                  6                  0.0
IpForwDatagrams                 0                  0.0
IpInUnknownProtos               0                  0.0
IpInDiscards                    0                  0.0
IpInDelivers                    5198901            0.0
IpOutRequests                   5249702            0.0
IpOutDiscards                   42                 0.0
IpOutNoRoutes                   0                  0.0
...
```

이들 중 커널 내부에서의 패킷 유실과 관련된 통계를 추려보면 다음과 같습니다.
```
IpInAddrErrors
IpInDiscards
IpOutDiscards
TcpInErrs
TcpInCsumErrors
UdpNoPorts
UdpInErrors
UdpRcvbufErrors
UdpSndbufErrors
UdpInCsumErrors
UdpMemErrors
Ip6InDiscards
Ip6OutDiscards
Udp6NoPorts
Udp6InErrors
Udp6RcvbufErrors
Udp6SndbufErrors
Udp6InCsumErrors
TcpExtIPReversePathFilter
TcpExtListenOverflows
TcpExtListenDrops
TcpExtTCPBacklogDrop
TcpExtTCPMinTTLDrop
TcpExtTCPTimeWaitOverflow
TcpExtTCPReqQFullDrop
TcpExtTCPOFODrop
TcpExtTCPRcvQDrop
TcpExtTCPZeroWindowDrop
...
```

각 통계들의 의미는 다른 포스트에서 더 깊게 다루겠습니다.

node_exporter에서는 `--collector.netstat` 옵션을 통해 수집할 수 있습니다. 이 옵션은 기본으로 켜져 있습니다.

그러나 메트릭이 너무 많기 때문에 모두 수집하지는 않고, `--collector.netstat.fields` 옵션에 매칭되는 메트릭들만 수집합니다.
node_exporter 버전 1.7.0 기준으로 --collector.netstat.fields 옵션의 기본값은 아래와 같습니다.
```text
"^(.*_(InErrors|InErrs)|Ip_Forwarding|Ip(6|Ext)_(InOctets|OutOctets)|Icmp6?_(InMsgs|OutMsgs)|TcpExt_(Listen.*|Syncookies.*|TCPSynRetrans|TCPTimeouts)|Tcp_(ActiveOpens|InSegs|OutSegs|OutRsts|PassiveOpens|RetransSegs|CurrEstab)|Udp6?_(InDatagrams|OutDatagrams|NoPorts|RcvbufErrors|SndbufErrors))$"
```
그리고 이 기본값으로 수집되는 패킷 유실 관련 메트릭들은 아래와 같습니다.
```text
node_netstat_TcpExt_ListenDrops
node_netstat_TcpExt_ListenOverflows
node_netstat_Tcp_InErrs
node_netstat_Udp6_RcvbufErrors
node_netstat_Udp6_SndbufErrors
node_netstat_Udp_InErrors
node_netstat_Udp_NoPorts
node_netstat_Udp_RcvbufErrors
node_netstat_Udp_SndbufErrors
```
이 기본값으로 수집되는 메트릭만 중요하고, 다른 메트릭은 덜 중요하다는 의미는 아닙니다. `--collector.netstat.fields` 옵션의 기본값은 node_exporter에 해당 옵션을 구현한 사람이 임의로 정하였고[^3], 그 이후 몇몇 사람들이 자신들이 원하는 메트릭을 추가해왔을 뿐입니다. 따라서 필요에 따라 해당 옵션을 적절히 수정하는 것이 좋아 보입니다.

## 트레이싱
이번에는 트레이싱을 통해 호스트에서의 패킷 유실을 모니터링하는 방법을 알아보겠습니다. 

커널에서 패킷을 드랍할 때에는 `kfree_skb` 함수가 호출되어 패킷을 메모리에서 해제합니다. 따라서 이 함수를 트레이싱하면 메트릭을 보는 것보다 더 자세한 상황을 알 수 있습니다.

### 예시
패킷 유실이 발생하는 상황을 임의로 만들고, ftrace를 사용하여 `kfree_skb` 함수를 트레이싱하여 원인을 파악해 보겠습니다. 아래 명령어를 통해 ftrace로 `kfree_skb` tracepoint 이벤트를 추적합니다.
```bash
echo kfree_skb > /sys/kernel/tracing/set_event
echo 1 > /sys/kernel/tracing/stacktrace
```
```
cat /sys/kernel/tracing/trace
          <idle>-0       [001] ..s. 68117391.627057: kfree_skb: skbaddr=0000000093c2ae62 protocol=2048 location=000000007e36f1a2
          <idle>-0       [001] ..s. 68117391.627062: <stack trace>
 => kfree_skb
 => ip_error
 => ip_sublist_rcv_finish
 => ip_sublist_rcv
 => ip_list_rcv
 => __netif_receive_skb_list_core
 => netif_receive_skb_list_internal
 => gro_normal_list.part.0
 => napi_complete_done
 => gro_cell_poll
 => net_rx_action
 => __do_softirq
 => irq_exit
 => do_IRQ
 => ret_from_intr
 => native_safe_halt
 => arch_cpu_idle
 => default_idle_call
 => do_idle
 => cpu_startup_entry
 => start_secondary
 => secondary_startup_64
 ...
```
`ip_sublist_rcv_finish` 함수에서 `ip_error` 함수를 호출했고, `ip_error` 함수에서 `kfree_skb`를 호출한 것을 볼 수 있습니다. 하지만 아직은 정확히 어떤 이유로 패킷 유실이 발생했는지는 알기 어렵습니다.

이번에는 ftrace의 function_graph tracer를 사용하여 전후 상황을 좀 더 살펴보겠습니다.
```bash
# 이전 트레이싱 설정 정리
echo > /sys/kernel/tracing/set_event
echo 0 > /sys/kernel/tracing/stacktrace

# "ip"로 시작하는 함수들과 kfree_skb 함수를 트레이싱
echo "ip*" > /sys/kernel/tracing/set_ftrace_filter
echo kfree_skb >> /sys/kernel/tracing/set_ftrace_filter
echo function_graph > /sys/kernel/tracing/current_tracer
echo 1 > /sys/kernel/tracing/tracing_on
```

```bash
cat /sys/kernel/tracing/trace
 1)               |  ip_list_rcv() {
 1)   0.319 us    |    ip_rcv_core.isra.0();
 1)               |    ip_sublist_rcv() {
 1)               |      iptable_mangle_hook [iptable_mangle]() {
 1)   0.426 us    |        ipt_do_table [ip_tables]();
 1)   0.879 us    |      }
 1)               |      ip_rcv_finish_core.isra.0() {
 1)               |        ip_route_input_noref() {
 1)               |          ip_route_input_rcu() {
 1)   1.612 us    |            ip_route_input_slow();
 1)   2.090 us    |          }
 1)   2.537 us    |        }
 1)   4.133 us    |      }
 1)               |      ip_sublist_rcv_finish() {
 1)               |        ip_error() {
 1) + 36.817 us   |          kfree_skb();
 1) + 37.410 us   |        }
 1) + 37.882 us   |      }
 1) + 43.969 us   |    }
 1) + 45.027 us   |  }
 ...
```
`ip_sublist_rcv_finish` 함수가 실행되기 전에 `ip_route_input_slow` 함수가 실행된 것을 볼 수 있습니다. `ip_route_input_slow`를 살펴보면 `IN_DEV_FORWARD`가 거짓이면 `EHOSTUNREACH`를 반환하고 `dst.input`에 `ip_error` 함수 포인터를 저장하고 있습니다.
```c
static int ip_route_input_slow(struct sk_buff *skb, __be32 daddr, __be32 saddr,
			       u8 tos, struct net_device *dev,
			       struct fib_result *res)
{
    // ...
    if (!IN_DEV_FORWARD(in_dev))
      err = -EHOSTUNREACH;
    goto no_route;
    // ...
    if (res->type == RTN_UNREACHABLE) {
        rth->dst.input= ip_error;
        rth->dst.error= -err;
        rth->rt_flags 	&= ~RTCF_LOCAL;
    }
    // ...
}
```
`ip_forward` 커널 파라미터를 확인해 보니 포워딩 기능이 꺼져있었습니다.
```bash
$ sysctl -a | grep ip_forward
net.ipv4.ip_forward = 0
```
해당 호스트가 포워딩을 해야 한다면 이 옵션을 켜두어야 하고, 그렇지 않다면 상단 라우터의 잘못된 라우팅으로 인해 엉뚱한 패킷이 들어오고 있거나, 호스트의 IP 설정이 누락된 것일 수도 있습니다. 이 경우엔 IP 설정이 누락된게 문제였습니다. 이렇게 `kfree_skb` 함수를 트레이싱하면 패킷 드랍 원인을 조금 더 자세히 파악해 볼 수 있습니다.

### kfree_skb 트레이싱의 한계
그러나 이러한 `kfree_skb` 트레이싱은 `kfree_skb` tracepoint가 제공해 주는 정보가 제한적이라는 한계가 있습니다. 커널 5.17 미만 버전에서는 skb의 메모리 주소 정도만 알려주는데, 이 정보만으로는 분석이 어렵습니다.

커널 5.17 버전부터는 패킷이 유실된 원인을 알려주는 DROP_REASON 필드가 추가되었습니다.[^4] DROP_REASON 필드 값들은 점점 추가되고 있으며, 커널 6.2 버전에서는 67개의 DROP_REASON 값을 제공하고 있습니다.[^5] 아래는 커널 6.2 버전에서 `kfree_skb` tracepoint의 출력 예시입니다. 존재하지 않는 소켓으로 패킷이 들어와 버려진 걸 알 수 있습니다.
```text
curl-58273   [000] ..s1. 2543693.602232: kfree_skb: skbaddr=... protocol=2048 location=... reason: NO_SOCKET
```
 
`kfree_skb` 트레이싱으로는 유실된 패킷의 IP 주소 정보를 알아내기가 어렵다는 한계도 있습니다. `kfree_skb` tracepoint에서는 IP 주소 정보를 제공해 주지 않는데, 그 이유는 `skb` 구조체에는 IP 외 ICMP, ARP 등 다른 프로토콜 패킷이 담길 수도 있기 때문입니다. Tracepoint 대신 kprobe를 사용하여 `kfree_skb` 함수의 인자인 skb를 직접 추적하는 것도 어렵습니다. `kfree_skb`가 호출되면 곧바로 skb가 메모리에서 해제되기 때문입니다.

만일 추적하려는 패킷 유실이 TCP 계층에서 발생하는 거라면, `kfree_skb` 대신 `tcp_drop` 함수를 트레이싱하여도 됩니다. 패킷이 유실되더라도 소켓은 유지되기에 `sock` 구조체를 통해 유실된 패킷의 엔드포인트 IP 주소를 확인할 수 있습니다. 하지만 TCP 계층이 아닌 곳에서 패킷이 유실되는 상황은 `tcp_drop` 함수로 추적할 수 없다는 점과, `tcp_drop`은 아직 tracepoint에 추가되지 않았기 때문에 불안정한 API라는 점을 유의해야 합니다.

## 정리
지금까지 리눅스에서 패킷 유실을 모니터링하는 방법들에 대해 알아보았습니다.

리눅스의 네트워킹 스택이 워낙 방대하여 이 글에서 미처 다루지 못한 것들도 많습니다. 처음부터 모든 메트릭을 다 이해하고 모니터링하는 것 보다는 실제로 문제들을 겪으면서 경험적으로 추가해가는 것도 좋은 방법일 것 같습니다.

다음 글에서는 호스트 외부의 네트워크의 문제를 리눅스에서 모니터링하는 방법에 대해 소개드리겠습니다.

[^1]: [https://www.kernel.org/doc/html/v5.15/networking/statistics.html#struct-rtnl-link-stats64](https://www.kernel.org/doc/html/v5.15/networking/statistics.html#struct-rtnl-link-stats64){:target="_blank"}
[^2]: [https://www.kernel.org/doc/html/v5.15/networking/scaling.html](https://www.kernel.org/doc/html/v5.15/networking/scaling.html){:target="_blank"}
[^3]: [https://github.com/prometheus/node_exporter/pull/876](https://github.com/prometheus/node_exporter/pull/876){:target="_blank"}
[^4]: [https://github.com/torvalds/linux/commit/c504e5c2f9648a1e5c2be01e8c3f59d394192bd3](https://github.com/torvalds/linux/commit/c504e5c2f9648a1e5c2be01e8c3f59d394192bd3){:target="_blank"}
[^5]: [https://github.com/torvalds/linux/blob/v6.2/include/net/dropreason.h](https://github.com/torvalds/linux/blob/v6.2/include/net/dropreason.h){:target="_blank"}
[^6]: [https://arthurchiao.art/blog/tcp-retransmission-may-be-misleading/](https://arthurchiao.art/blog/tcp-retransmission-may-be-misleading/){:target="_blank"}
