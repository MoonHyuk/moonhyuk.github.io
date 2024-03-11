---
title: 타이머 기반 유휴 커넥션 정리 시 주의점
tags:
  - HTTP
  - TCP/IP
---

## 유휴 커넥션이란?
커넥션 기반 통신을 위해서는 각 엔드포인트 서버에서 커넥션의 정보를 저장하고 있어야 합니다. 커넥션 정보에는 해당 커넥션의 상태(established, syn_sent, time_wait 등), 엔드포인트의 IP 주소와 포트 번호, TCP 타이머 정보 등이 있습니다. 

만약 커넥션을 통해서 더 이상 데이터를 주고받지 않는데도 불구하고 오랫동안 유지된다면 서버 리소스를 낭비하게 됩니다. 많은 HTTP 어플리케이션에서는 이런 유휴 커넥션을 줄이기 위해 일정 시간 동안 활동이 없거나, 일정 수의 요청을 처리한 커넥션들을 종료하는 설정을 제공합니다. 
예를 들어 Nginx에서는 keepalive_timeout[^1] 값을 통해 유휴 커넥션 지속 시간을 설정할 수 있습니다. 이 글에서는 이 설정을 idle timeout이라고 부르겠습니다.

하지만 idle timeout을 사용하여 타이머 기반으로 유휴 커넥션을 정리할 때는 주의하지 않으면 네트워크 오류가 발생할 수 있습니다. 오류가 발생할 수 있는 몇 가지 상황들을 알아보겠습니다.

## 타이머 기반 유휴 커넥션 정리 시 주의점
### 1. 클라이언트와 서버가 직접 통신하는 경우
아래 그림처럼 클라이언트와 서버가 직접 통신하는 경우에는 클라이언트의 idle timeout 값을 서버의 idle timeout 값 보다 작게 설정해야 합니다.

![2024030801.png](/assets/images/2024030801.png){:width="500"}

그렇지 않으면 클라이언트에서 요청 전송 시 커넥션 오류가 발생할 수 있습니다. 예를 들어 클라이언트의 idle timeout이 60초, 서버의 idle timeout이 10초라고 해보겠습니다. 아래는 클라이언트가 서버와 커넥션을 맺고 요청과 응답을 주고받은 후의 모습입니다.  

![2024030802.png](/assets/images/2024030802.png){:width="500"}

클라이언트와 서버에는 각각 커넥션이 생성되었고 커넥션이 유휴 상태로 지속될 경우 클라이언트에서는 60초 후에, 서버에서는 10초 후에 정리하려 합니다. 그리고 딱 10초 후에 클라이언트가 추가 요청을 보냈다고 해보겠습니다. 

![2024030803.png](/assets/images/2024030803.png){:width="500"}

클라이언트에서는 아직 커넥션의 생명이 50초 남아있으므로 해당 커넥션을 재사용하여 요청을 전송했습니다. 하지만 그와 동시에 서버에서는 커넥션이 만료되어 정리하고 클라이언트에게 FIN 혹은 RST을 전송합니다. 이 경우 클라이언트는 정상 응답을 받지 못하고 오류가 발생합니다.

이 상황을 간단히 재현해 보겠습니다. Nginx 서버에 keepalive_timeout 값을 1초로 설정하고 Python 클라이언트에서는 커넥션을 유지한 채 약 1초마다 요청을 전송하도록 했습니다.
```conf
# nginx.conf

http {
    keepalive_timeout 1;
}
```
```python
# client.py

import random
import time

import requests

s = requests.Session()

while True:
    r = s.get('http://myserver.com')
    print(r.status_code, time.time())
    time.sleep(random.uniform(0.9, 1.0))
```

Python 스크립트를 실행해 보면 확률적으로 `RemoteDisconnected` 오류가 발생하는 걸 볼 수 있습니다.

```shell
$ python3 client.py
200 1690337818.580432
200 1690337819.544884
Traceback (most recent call last):
...
    raise RemoteDisconnected("Remote end closed connection without"
http.client.RemoteDisconnected: Remote end closed connection without response
...
```

### 2. Layer 7 로드밸런서를 사용하는 경우
위에서 설명드린 문제는 커넥션을 재사용하는 모든 클라이언트에서 발생합니다. 대부분의 Layer 7 로드밸런서는 성능 이점을 위해 서버와의 커넥션을 유지하며 재사용합니다. 또, 로드밸런서도 서버에게는 클라이언트 역할을 하기 때문에 로드밸런서의 idle timeout 값도 서버의 idle timeout 값 보다 작아야 합니다.

클라이언트와 로드밸런서 그리고 서버의 idle timeout 권장 설정을 그림으로 나타내면 다음과 같습니다. 

![2024030804.png](/assets/images/2024030804.png){:width="700"}

AWS와 GCP 등 클라우드 벤더에서도 Layer 7 로드밸런서인 Application Load Balancer 제품 사용 시 로드밸런서의 idle timeout이 서버의 idle timeout보다 작게 설정하는 것을 권장하고 있습니다.

> The load balancer's backend keepalive timeout should be less than the keepalive timeout used by software running on your backends. This avoids a race condition where the operating system of your backends might close TCP connections with a TCP reset (RST). 
> 
> \- "External Application Load Balancer overview", Google Cloud [^2] 

>  We also recommend that you configure the idle timeout of your application to be larger than the idle timeout configured for the load balancer. Otherwise, if the application closes the TCP connection to the load balancer ungracefully, the load balancer might send a request to the application before it receives the packet indicating that the connection is closed. If this is the case, then the load balancer sends an HTTP 502 Bad Gateway error to the client.
> 
> \- "Application Load Balancers", AWS [^3]

### 3. Stateless 로드밸런서를 사용하는 경우
높은 확장성과 가용성을 가진 로드밸런서 아키텍처를 만들기 위해 stateless 로드밸런서를 사용하기도 합니다. 이에 대한 자세한 내용은 Cloudflare의 블로그[^4]와 Vincent Bernat의 글[^5]에서 자세히 설명하고 있습니다.

로드밸런서가 stateless하기 위해서는 아래 조건들을 만족해야 합니다.  

1. 로드밸런싱 방식은 source IP 혹은 source IP + source port consistence hashing을 사용합니다.
2. 서버의 응답 패킷이 로드밸런서를 통하지 않도록 합니다. (Direct Server Return)
3. 로드밸런서에서는 TCP 동작을 전혀 하지 않아야 합니다. 예를 들어 로드밸런서 스스로 클라이언트나 서버로 FIN 혹은 RST을 전송하면 안됩니다.

Stateless 로드밸런서에서도 커넥션을 기록하기는 합니다. 이 커넥션은 TCP 동작을 위한 것이 아니라, 이미 연결이 있는 통신에 대해서는 이후 패킷들도 동일한 백엔드 서버로 전달하기 위한 것입니다. 만약 stateless 로드밸런서에서 커넥션 정보가 없다면 백엔드 서버가 추가 혹은 삭제될 때 consistence hashing 결과가 달라져 TCP 상태가 깨질 수 있습니다.

Stateless 로드밸런서에도 커넥션이 있으므로 유휴 커넥션을 정리하는 idle timeout 설정을 제공합니다. 하지만 idle timeout으로 커넥션이 정리되더라도 stateless 로드밸런서는 클라이언트나 서버로 FIN 혹은 RST을 전송하지 않습니다. 단지 자신의 커넥션 테이블에서만 조용히 삭제하고, 클라이언트가 이후 요청을 보낸다면 consistence hashing을 통해 백엔드 서버를 선택하고 다시 커넥션을 생성합니다.

따라서 stateless 로드밸런서에서는 idle timeout에 관한 제약 조건이 없습니다.

### 4. 올바르지 않은 Stateless 로드밸런서를 사용하는 경우
앞서 stateless 로드밸런서는 3가지 조건을 만족해야 한다고 설명드렸습니다. 만약 설정 오류로 인해 3가지 조건들 중 하나를 만족시키지 못한다면 어떻게 동작할지 알아보겠습니다.

우선 로드밸런싱 방식을 consistence hashing이 아니라 라운드로빈으로 설정했을 경우입니다. 클라이언트와 서버의 idle timeout은 60초, 로드밸런서의 idle timeout은 10초라고 가정하겠습니다. 클라이언트가 커넥션을 생성하고 요청을 보내면 커넥션 상태는 다음과 같습니다.

![2024030805.png](/assets/images/2024030805.png){:width="700"}

이때 로드밸런서의 커넥션 `Connection 1'`에는 Client로부터 온 패킷은 Server 1로 전달하라는 내용이 저장됩니다. 그리고 10초 동안 클라이언트가 요청을 보내지 않아 로드밸런서에서는 idle timeout으로 인해 유휴 커넥션이 정리됩니다.

![2024030806.png](/assets/images/2024030806.png){:width="700"}

이후 곧바로 클라이언트가 요청을 다시 보내면, 로드밸런서에는 커넥션 정보가 없고 로드밸런싱 방식이 라운드로빈이므로 이번에는 패킷을 server 2로 전달해주게 됩니다. 하지만 server 2는 해당 클라이언트와 커넥션을 맺지 않았기 때문에 RST을 응답합니다.

![2024030807.png](/assets/images/2024030807.png){:width="700"}

이렇게 클라이언트는 오류를 수신하게 됩니다.

이번에는 로드밸런서가 불완전한 TCP 동작을 하는 상황을 살펴보겠습니다. 위와 동일하게 클라이언트가 한번 요청을 보낸 후 10초가 지나 로드밸런서에서는 유휴 커넥션이 정리되었다고 해보겠습니다.

![2024030806.png](/assets/images/2024030806.png){:width="700"}

그리고 예를 들어 로드밸런서가 자신에게 정보가 없는 커넥션으로 패킷이 들어올 때는 SYN 플래그가 켜진 세그먼트만 허용하고 나머지 경우에는 RST을 응답하는 잘못된 TCP 동작을 하게 된다면, 클라이언트가 보낸 요청에 대해 로드밸런서가 RST 응답을 할 수 있습니다.   

![2024030808.png](/assets/images/2024030808.png){:width="700"}

실제로 여러 하드웨어 로드밸런서는 기본적으로 이처럼 동작하기에, 커넥션 정보가 없을 때 SYN이 아닌 세그먼트도 허용하도록 명시적으로 설정을 해주어야 합니다. (F5에서는 `loose initialization`을 `enabled`로 설정하고, Citrix에서는 `connfailover`를 `stateless`로 설정합니다.) 

Stateless 로드밸런서가 잘못 설정되어 있다면 로드밸런서의 idle timeout이 서버나 클라이언트의 idle timeout보다 작을 때 문제가 발생할 수 있습니다. 이때는 idle timeout을 변경하기보다는 로드밸런서를 stateless 하게 설정해 주는 것이 좋아 보입니다. 

[^1]: [https://nginx.org/en/docs/http/ngx_http_core_module.html#keepalive_timeout](https://nginx.org/en/docs/http/ngx_http_core_module.html#keepalive_timeout){:target="_blank"}
[^2]: [https://cloud.google.com/load-balancing/docs/https](https://cloud.google.com/load-balancing/docs/https){:target="_blank"}
[^3]: [https://docs.aws.amazon.com/elasticloadbalancing/latest/application/application-load-balancers.html#connection-idle-timeout](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/application-load-balancers.html#connection-idle-timeout){:target="_blank"}
[^4]: [https://blog.cloudflare.com/high-availability-load-balancers-with-maglev/](https://blog.cloudflare.com/high-availability-load-balancers-with-maglev/){:target="_blank"}
[^5]: [https://vincent.bernat.ch/en/blog/2018-multi-tier-loadbalancer](https://vincent.bernat.ch/en/blog/2018-multi-tier-loadbalancer){:target="_blank"}
