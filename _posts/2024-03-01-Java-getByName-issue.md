---
title: Java InetAddress DNS 질의 성능 문제
tags:
  - Java
  - DNS
---

## 개요
멀티스레딩 Java 어플리케이션에 트래픽이 평소보다 조금 더 몰려 요청을 제대로 처리하지 못하는 장애가 발생했습니다. 하지만 장애 당시 트래픽은 어플리케이션에서 예상하던 최대 수용 가능한 트래픽에 한참 못 미치는 수준이었습니다.

장애 원인을 분석해 보니 특정 조건을 만족할 때 DNS 질의가 성능 문제를 일으킴을 확인했습니다.

## 문제 발생 조건
문제가 발생하는 조건은 다음과 같습니다.
1. DNS 캐시 TTL이 0초로 설정되어 있습니다.
2. 멀티스레딩을 사용하고 있습니다.
3. 어플리케이션에서 다른 서비스 도메인을 질의합니다 (지속 커넥션을 사용하지 않아 매 요청마다 DNS 질의가 발생한다면 상황이 훨씬 악화됩니다).
4. 어플리케이션에서 사용하는 네트워크 클라이언트 라이브러리가 DNS 질의를 위해 `InetAddress`을 사용합니다.

위 조건들을 모두 만족할 때 두 가지 성능 문제가 발생합니다.

아래는 문제가 발생하는 상황을 재현한 코드입니다. 스레드 수 `num_threads`를 입력으로 받아 그 수만큼 스레드를 만들고, 각 스레드에서 DNS 질의를 나눠서 처리합니다. DNS 캐싱을 하지 않도록 `networkaddress.cache.ttl`을 `0`으로 설정했습니다.

```java
import java.net.InetAddress;

public class Main {
    public static void main(String[] args) {
        java.security.Security.setProperty("networkaddress.cache.ttl", "0");

        int num_queries = 84000;
        int num_threads = Integer.parseInt(args[0]);
        System.out.println("num_threads: " + args[0]);

        int num_queries_per_thread = num_queries / num_threads;
        for (int i = 0; i < num_threads; i++) {
            Thread t = new Thread(new LookupThread(num_queries_per_thread));
            t.start();
        }
    }
}

class LookupThread implements Runnable {
    private final int num_queries;

    public LookupThread(int num_queries) {
        this.num_queries = num_queries;
    }

    public void run() {
        for (int i = 0; i < this.num_queries; i++) {
            try {
                InetAddress.getByName("myservice.com");
            } catch (Exception e) {
                System.out.println(e);
            }
        }
    }
}
```

## 두가지 문제

### 1. 동일 호스트 질의 시 락 발생
우선 클라이언트로부터 요청을 받으면 매번 다른 서비스를 호출하고, 그 결과를 전달해 주는 어플리케이션이 있다고 해보겠습니다. 만약 이 어플리케이션이 코어가 10개 이상인 서버에서 10개 스레드로 동작한다고 하면, 아래 그림처럼 동작하기를 기대합니다.

![Java-inetaddress-perf-image1.png](/assets/images/Java-inetaddress-perf-image1.png)

문제를 단순하게 보기 위해 다른 서비스 호출은 동기 블로킹으로 실행된다고 가정하고, DNS 질의와 다른 서비스 요청은 모두 1ms가 소요된다고 가정하였습니다.

클라이언트의 요청 하나는 DNS 질의 시간 + 다른 서비스 API 응답 대기 시간인 `2ms`에 처리되므로 1개 스레드에서 초당 500개의 요청을 처리할 수 있고, 10개 스레드로는 초당 5,000개의 요청을 처리할 수 있습니다.

하지만 실제 상황에서는 초당 요청이 약 1,000개를 넘을 때부터 어플리케이션이 처리를 못하기 시작했고, 응답 시간이 느려지며 장애가 발생했습니다.

문제 원인을 파악하기 위해 tcpdump를 실행하였는데, DNS 질의 부분에서 특이한 점을 발견했습니다.

![Java-inetaddress-perf-image2.png](/assets/images/Java-inetaddress-perf-image2.png)

위 사진에서 주황색 네모는 DNS 질의 패킷을, 연두색 네모는 DNS 응답 패킷을 나타냅니다. 분명 요청을 10개 스레드에서 병렬로 처리하고 있는데, DNS 질의/응답은 한 번에 하나씩만 동작하고 있는 걸로 보입니다.

만약 DNS 질의가 여러 스레드에서 동시에 진행되었다면 아래 사진처럼 동시에 여러 개의 DNS 질의가 발생하는 모습이 보였을 겁니다.

![Java-inetaddress-perf-image9.png](/assets/images/Java-inetaddress-perf-image9.png)


Java의 DNS 질의 동작에 어떤 문제가 있다고 생각하고, DNS 관련 로직을 살펴보았습니다. Java의 DNS 질의는 `InetAddress`의 `getByName` 메소드를 통해 실행됩니다. openjdk 19 버전 기준으로, `getByName`의 플로우는 다음과 같습니다.

![Java-inetaddress-perf-image3.png](/assets/images/Java-inetaddress-perf-image3.png)

`NameServiceAddresses`은 DNS 질의를 보내고, 어플리케이션이 DNS 캐싱을 하도록 설정했다면 응답을 `CachedAddresses` 객체에 넣고 이를 캐시에 저장하는 역할을 합니다. 응답이 캐싱된 이후에는 `CachedAddresses`에서 값을 바로 읽습니다.

어플리케이션이 DNS 캐시를 사용하지 않도록 설정했다면 `CachedAddresses` 객체가 캐시에 저장되지 않고, 매번 `NameServiceAddresses` 객체를 생성하게 됩니다. 문제는 `NameServiceAddresses`의 `get` 메소드가 질의된 호스트 별로 락을 건다는 점입니다.

```java
private static final class NameServiceAddresses implements Addresses {
    private final String host;
    private final ReentrantLock lookupLock = new ReentrantLock();

    NameServiceAddresses(String host) {
        this.host = host;
    }

    @Override
    public InetAddress[] get() throws UnknownHostException {
        Addresses addresses;
        // only one thread is doing lookup to name service
        // for particular host at any time.
        lookupLock.lock();
        try {
            // ...
        } finally {
            lookupLock.unlock();
        }
        return addresses.get();
```

이 락은 여러 스레드가 동시에 동일 호스트를 질의할 때, 한 스레드에서만 DNS 질의를 수행하고 캐시에 값을 넣게 하기 위함인 것으로 보입니다. 하지만 DNS 응답을 캐싱하지 않도록 설정했음에도 락이 걸리는 것은 아이러니합니다. 심지어 캐싱을 하지 않게 설정하면 모든 스레드에서 매번 `NameServiceAddresses.get` 메소드가 호출되므로, DNS 질의는 한 번에 하나의 스레드에서만 진행이 되고 다른 스레드에서는 대기를 하게 됩니다.

즉, 실제로 10개 스레드에서 요청을 처리하는 모습은 아래 그림과 같습니다.

![Java-inetaddress-perf-image4.png](/assets/images/Java-inetaddress-perf-image4.png)

이렇게 되면 한 어플리케이션의 성능은 DNS 응답 대기 시간에 크게 영향을 받습니다. 만약 DNS 응답 시간이 평균 1ms라면, 한 프로세스에서 처리 가능한 요청은 초당 1,000개를 넘지 못합니다. DNS 응답이 단 1ms만 느려져도 초당 처리 가능한 요청 수는 절반으로 감소합니다.

### 2. Race condition 문제
위 문제 말고도 race condition으로 인해 DNS 응답이 느려지는 문제도 있습니다.

문제 상황을 재현한 코드에서 `num_threads`를 1로 하여 실행하고, async-profiler로 실행된 함수들을 트레이싱 해봤습니다.

![Java-inetaddress-perf-image6.png](/assets/images/Java-inetaddress-perf-image6.png)

아래는 `num_threads`가 8일 때 트레이싱 결과입니다.

![Java-inetaddress-perf-image5.png](/assets/images/Java-inetaddress-perf-image5.png)

`num_threads`가 1일 때와는 달리 우뚝 솟아난 부분들이 보이는데, 이를 확대하여 보면 이렇습니다.

![Java-inetaddress-perf-image7.png](/assets/images/Java-inetaddress-perf-image7.png){: width="400"}

`InetAddress$NameServiceAddresses.get` 메소드가 재귀적으로 호출된 걸 볼 수 있습니다. 정상 상황이라면 `NameServiceAddresses.get` 메소드에서는 `InetAddress.getAddressesFromNameService` 메소드를 호출한 후 종료되어야 합니다.

이런 재귀 호출이 발생하는 이유는 `NameServiceAddresses.get` 메소드에서 race condition으로 인해 현재 캐시에 저장된 Addresses 객체가 본인의 것과 다르면 다시 `Addresses.get`을 호출하기 때문입니다. DNS 캐싱을 하지 않는 상황이라면 `Addresses` 객체의 타입은 항상 `NameServiceAddresses` 이므로, `NameServiceAddresses.get`가 다시 호출됩니다.

아래 빨간색 플로우가 반복적으로 실행되는 것입니다.

![Java-inetaddress-perf-image8.png](/assets/images/Java-inetaddress-perf-image8.png){: width="550"}

Race condition이 발생하는 상황을 예를 들어보겠습니다.
1. Thread 1에서 `InetAddress.getByName("myservice.com")`이 실행됩니다.
2. 캐시에 `myservice.com`이 없으므로, Thread 1이 `NameServiceAddresses` 객체 `NS1`를 생성하고 캐시에 저장합니다.
3. Thread 1에서 `NameServiceAddresses.get`이 실행되고, Thread 1이 락을 획득합니다.
4. Thread 1에서 캐시에 있는 Addresses 객체가 본인 것과 동일한지 다시 확인합니다. 확인 후 `myservice.com`을 질의를 하고, 응답을 기다립니다.
5. Thread 2에서 `InetAddress.getByName("myservice.com")`이 호출됩니다.
6. 캐시에 `myservice.com`이 있습니다. Thread 2는 Thread 1이 만든 객체 `NS1`을 캐시에서 읽어옵니다.
7. Thread 2가 `NS1.get`을 실행합니다. `NS1`의 타입은 `NameServiceAddresses` 이므로 `NameServiceAddresses.get`이 실행됩니다. 하지만 `Thread 1`이 락을 걸고 있으므로 잠시 대기합니다.
8. Thread 1에서 DNS 응답을 받았고, DNS 캐싱을 하지 않도록 설정하였기 때문에 캐시에서 `myservice.com`을 삭제합니다. Thread 1은 `myservice.com`의 레코드를 반환하고 락을 해제합니다.
9. Thread 2가 락을 획득합니다.
10. 그와 동시에 Thread 1에서 다시 `InetAddress.getByName("myservice.com")`이 실행됩니다. 이 때 캐시에는 `myservice.com`이 없습니다. 따라서 Thread 1이 `NameServiceAddresses` 객체 `NS2`를 생성하고 캐시에 저장합니다.
11. Thread 2는 캐시에 있는 Addresses 객체가 본인 것과 동일한지 다시 확인합니다. Thread 2가 가진 것은 `NS1`이지만, 캐시에 있는 것은 `NS2`입니다. 서로 다르므로 Thread 2는 락을 풀고 `NS2.get`을 실행합니다. `NS2`의 타입도 `NameServiceAddresses` 이므로 `NameServiceAddresses.get`이 다시 실행됩니다.
12. Thread 1이 락을 획득합니다. 8번으로 돌아가 반복됩니다...

이 Race condition이 성능에 얼마나 영향을 미칠지 알아보기 위해 스레드 수를 바꿔가며 `InetAddress.getByName`의 성능을 측정해 봤습니다.

![Java-inetaddress-perf-image10.png](/assets/images/Java-inetaddress-perf-image10.png){: width="550"}
<center>[스레드가 1개일 때 InetAddress.getByName 실행 시간 분포]</center>

스레드가 1개일 때는 99 백분위수가 1ms 내외였고, 99.9 백분위수는 5ms 내외였습니다.

![Java-inetaddress-perf-image11.png](/assets/images/Java-inetaddress-perf-image11.png){: width="550"}
<center>[스레드가 2개일 때 InetAddress.getByName 실행 시간 분포]</center>

반면 race condition이 발생할 수 있는 2개 스레드에서는 99 백분위가 10ms 내외, 99.9 백분위는 75ms까지 증가했습니다. 심지어 아주 운이 나쁜 경우에는 DNS 질의 시간이 1초까지 증가하기도 했습니다.
이렇게 낮은 확률로 `NameServiceAddresses.get`이 불필요하게 여러 차례 실행되면 DNS 질의 성능이 나빠질 수 있습니다.

## 해결 방법
이 문제를 완화하는 가장 쉬운 방법은 발생 조건을 제거하는 것입니다.

즉,
1. Java에서 DNS 캐싱을 활성화한다.
2. 멀티스레딩 대신 멀티프로세싱을 사용한다.
3. 커넥션을 재사용하여 동일 도메인에 대한 DNS 질의를 줄인다.
4. DNS 리졸빙을 위해 `InetAddress`의 를 쓰지 않는 클라이언트 라이브러리를 사용한다.

등의 방법들이 있습니다.

하지만 근본적인 해결을 위해서는 Java에서 DNS 캐시 TTL을 0으로 설정했을 때 `NameServiceAddresses`에서 불필요한 락을 걸거나 캐시된 객체를 확인하는 등의 동작을 하지 않도록 `InetAddress` 코드가 수정되어야 합니다.
