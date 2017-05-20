# Performance problem with Kubernetes-based BIND container

| Title | Value |
| :--: | :---:|
| Author | [Heechul Kim](mailto:jijisa@iorchard.net) |
| Date | 2017.05.20 |
[//]: # "| Revision | 201Y.MM.DD, 201Y.MM.DD | - for later use."


[[_TOC_]]

kubernetes 기반 BIND DNS container 서비스 프로젝트를 하고 있다.

성능 상 문제가 있어 여기에 기술한다.

## 문제 원인

BIND DNS 서비스는 가벼운 서비스이기 때문에 kubernetes 기반으로 빠르고(agile)
탄력적인(elastic) 서비스를 하기에 적합하다고 보았다.

그러나 성능면에서 문제가 발생하였다.

성능 실험은 다음과 같이 한다.
L4에 DNS container의 external ip를 등록하고 실제 트래픽을 흐르게 한다.
그리고 health check program은 주기적으로 100개의 query를 던져 몇 개가
응답하는 지 본다.
100개 다 응답하면 healthy한 것이고, 하나라도 응답이 안되면 상용 서비스 적용
실패로 간주한다.
L4에서 DNS container에 가중치를 증가시키면서 requests/seconds (rps)의 한계를
시험한다.

성능 실험 결과는 다음과 같이 나왔다.

실제 DNS 트래픽을 흘려 보면, 1개의 container가 적정 1000 ~ 1200 rps 정도까지만
처리가 가능하다. 2500 rps를 흘리면, health check 실패가 나온다.
물리 서버로 시험하면 10000-12000 rps 가 가능하다고 하니 차이가 크다.

수작업으로 넣은 iptables 는 성능이 잘 나온다. (DNAT 룰 적용)
그러나 kube-proxy가 넣은 iptables는 성능이 잘 나오지 않는다.
뭐가 다른가 분석이 필요하다.
그래서 수사망은 iptables의 성능 문제로 좁혀졌다.

## 실험 1. kubedns가 영향을 주는가?

먼저 클러스터 내부에 서비스하는 kubedns가 영향을 주는 것이 아닌가 하는 의심이
있어 확인을 위해 다음과 같이 실험했다.

BIND container를 하나만 켜고, 외부에서 cnn.com 을 query해 보았다.

외부 서버$ dig @<bind_svc_external_ip> cnn.com

다음은 컨테이너가 물린 virtual nic interface의 tcpdump 결과이다.
```
# tcpdump -n -i vethf83f700 udp port 53
13:58:16.615455 IP 10.100.13.1.46454 > 10.100.13.2.domain: 4528+ [1au] A? cnn.com. (36)
13:58:16.616294 IP 10.100.13.2.41548 > 192.41.162.30.domain: 46593 [1au] A? cnn.com. (48)
13:58:16.814478 IP 192.41.162.30.domain > 10.100.13.2.41548: 46593-| 0/7/2 (510)
13:58:17.247744 IP 10.100.13.2.38443 > 205.251.192.47.domain: 13685 [1au] A? cnn.com. (48)
13:58:17.250624 IP 205.251.192.47.domain > 10.100.13.2.38443: 13685*- 4/4/1 A 151.101.129.67, A 151.101.193.67, A 151.101.1.67, A 151.101.65.67 (236)
13:58:17.251015 IP 10.100.13.2.domain > 10.100.13.1.46454: 4528 4/4/8 A 151.101.65.67, A 151.101.1.67, A 151.101.193.67, A 151.101.129.67 (384)
```
위 dump에서 보이는 각 IP는 다음과 같이 해석된다.

    - 10.100.13.1 : docker0
    - 10.100.13.2 : bind container
    - 192.41.162.30 : l.gtld-servers.net (미국에 있는 .com 관리 네임서버)
    - 205.251.192.47 : ns-47.awsdns-05.com. (cnn.com의 네임서버)

위 dump를 보면 BIND가 자체적으로 recursive query를 수행하는 것을 볼 수 있다.

다시 한번 같은 query를 하면,
```
14:12:19.187171 IP 10.100.13.1.52585 > 10.100.13.2.domain: 32962+ [1au] A?
cnn.com. (36)
14:12:19.187725 IP 10.100.13.2.domain > 10.100.13.1.52585: 32962 4/4/8 A
151.101.1.67, A 151.101.129.67, A 151.101.65.67, A 151.101.193.67 (384)
```
바로 cache된 데이터를 리턴하는 것을 볼 수 있다.
kubedns는 BIND 서비스에 관여하지 않는다.


물론 kubedns를 켜고/끄고 성능 실험을 진행해 보았고
BIND container 성능과 kubedns는 무관함을 보았다.

결론: kubedns가 켜져 있든 꺼져 있든 결과에는 무관하다.


## 실험 2. docker0 트래픽 분석

이제 부하를 준 상태에서 docker0의 트래픽을 분석하기로 한다.
두번의 실험을 하는데, 한번은 kube-proxy가 만든 iptables(kube-proxy iptables라고 하자.) 기반으로 하고,
또 한번은 수작업으로 만든 iptables(manual DNAT iptables라고 하자.) 기반으로 하여 패킷 dump를 한다.

    # tcpdump -n -w dump.bin -i docker0 host 10.100.13.3

여기서  상이한 점을 발견할 수 있었다.

* kube-proxy iptables
```
16:26:45.785852 IP ares-k29.60862 > 10.100.13.3.domain: 3020+ AAAA?  cdn.megafile.co.kr. (36)
16:26:45.785869 IP ares-k29.36495 > 10.100.13.3.domain: 3765+ A? www.google.com. (32)
16:26:45.785871 IP ares-k29.51732 > 10.100.13.3.domain: 64486+ A? upushgw.uplus.co.kr. (37)
16:26:45.785889 IP ares-k29.54114 > 10.100.13.3.domain: 58931+ A? urs.microsoft.com. (35)
16:26:45.786002 IP 10.100.13.3.domain > 122.32.52.130.59329: 3765 16/4/4 A 210.92.119.24, A 210.92.119.30, A 210.92.119.40, A 210.92.119.39, A 210.92.119.49, A 210.92.119.25, A 210.92.119.44, A 210.92.119.54, A 210.92.119.45, A 210.92.119.20, A 210.92.119.59, A 210.92.119.34, A 210.92.119.50, A 210.92.119.55, A 210.92.119.35, A 210.92.119.29 (424)
16:26:45.786077 IP 10.100.13.3.domain > 49.168.157.153.54114: 58931 2/4/4 CNAME urs.microsoft.com.nsatc.net., A 40.74.131.199 (235)
16:26:45.786158 IP 10.100.13.3.59718 > 43.255.253.254.domain: 53404 AAAA? megagrid.stm.flexcdn.kr. (41)
16:26:45.786158 IP 10.100.13.3.56159 > ns2.bora.net.domain: 61786 [1au] A?  upushgw.uplus.co.kr. (48)
16:26:45.786612 IP ns2.bora.net.domain > 10.100.13.3.56159: 61786*- 1/2/3 A 106.103.255.208 (143)
16:26:45.786733 IP 10.100.13.3.domain > 49.166.115.71.51732: 64486 1/2/2 A 106.103.255.208 (132)
```

* manual DNAT iptables tcpdump 결과
```
17:42:46.733732 IP 182.225.81.191.53632 > 10.100.13.3.domain: 22076+ A? search.naver.com. (34)
17:42:46.733769 IP 125.191.163.58.50421 > 10.100.13.3.domain: 42077+ [1au] ANY? learnengs.com. (42)
17:42:46.733835 IP 112.153.78.17.45538 > 10.100.13.3.domain: 62723+ A? m.nate.com. (28)
17:42:46.733847 IP 182.215.65.22.64472 > 10.100.13.3.domain: 58869+ A? spcdnpc.i-mobile.co.jp. (40)
17:42:46.733944 IP 10.100.13.3.domain > 182.225.81.191.53632: 22076 3/3/3 CNAME search.naver.com.nheos.com., A 125.209.230.167, A 43.250.153.7 (205)
17:42:46.734003 IP ns-227.awsdns-28.com.domain > 10.100.13.3.58021: 42710*- 1/4/1 CNAME dmhleyyccs922.cloudfront.net. (220)
17:42:46.734026 IP 182.216.81.166.23063 > 10.100.13.3.domain: 62097+ A? logins.daum.net. (33)
17:42:46.734018 IP 10.100.13.3.domain > 112.153.78.17.45538: 62723 1/2/2 A 211.115.10.31 (112)
17:42:46.734028 IP 125.176.202.64.1028 > 10.100.13.3.domain: 36728+ A? uict2.ez-i.co.kr. (34)
17:42:46.734188 IP 10.100.13.3.domain > 125.176.202.64.1028: 36728 NXDomain 0/1/0 (92)
17:42:46.734206 IP 10.100.13.3.domain > 182.216.81.166.23063: 62097 3/2/2 CNAME logins.g.daum.net., A 180.70.134.231, A 180.70.134.230 (154)
```

차이가 보이는가?

kube-proxy iptables는 DNS query packet의 source ip가 항상 ares-k29이다.
즉, container가 구동 중인 호스트의 source ip로 변경된 것을 볼 수 있다.
manual DNAT iptables는 DNS query packet의 source ip가 실제 source ip로 유지된다.

여기서 얻은 정보는

    kube-proxy iptables는 들어오는 트래픽에 대해서 SNAT를 한다.

는 것이다.

물론 BIND container로부터 나가는 트래픽도 SNAT을 한다.

kube-proxy iptables를 보면, 들어오는 packet에 0x4 mark를 하고, 이 mark가
있는 패킷은 SNAT를 한다.


다음과 같이 요약하자.
* kube-proxy iptables

    - incoming dns query: SNAT -> DNAT
    - outgoing dns query/response: SNAT

outgoing dns query는 BIND container가 수행하는 recursive query를 의미한다.

* manual DNAT iptables

    - incoming dns query: DNAT only
    - outgoing dns query/response: SNAT

위 두가지 경우의 차이점은 결국 incoming dns query에서 kube-proxy는 SNAT을
한다는 점이다. 이것이 성능 차이를 유발할까?


## SNAT의 성능 문제

SNAT의 성능은 conntrack table과도 관련이 있다. netfilter는 SNAT을 위해
conntrack table를 이용한다. conntrack table은 Linked-list table로서 
memory에 만들어져 SNAT의 정보를 저장하기 위해 사용된다.

그렇다면, 혹시 conntrack table이 full이 되어 문제가 발생하는 것일까?
이것을 검증하기 위해 트래픽을 흘릴 때 conntrack table의 사용 개수를 저장해
보았다.
conntract table의 사용개수는 /proc/sys/net/netfilter/nf_conntrack_count로
볼 수 있다.

아래는 약 2500 rps 트래픽이 흐를 때 conntrack 개수 로그이다.
```
2017. 05. 19. (금) 13:52:34 KST :  33 / 4194304
2017. 05. 19. (금) 13:52:35 KST :  33 / 4194304
2017. 05. 19. (금) 13:52:36 KST :  149 / 4194304
2017. 05. 19. (금) 13:52:39 KST :  147 / 4194304
2017. 05. 19. (금) 13:52:40 KST :  266 / 4194304
2017. 05. 19. (금) 13:52:44 KST :  390 / 4194304
2017. 05. 19. (금) 13:52:45 KST :  466 / 4194304
2017. 05. 19. (금) 13:52:50 KST :  666 / 4194304
2017. 05. 19. (금) 13:53:09 KST :  1448 / 4194304
2017. 05. 19. (금) 13:53:13 KST :  1535 / 4194304
2017. 05. 19. (금) 13:53:20 KST :  1538 / 4194304
2017. 05. 19. (금) 13:53:24 KST :  1534 / 4194304
2017. 05. 19. (금) 13:53:25 KST :  1534 / 4194304
2017. 05. 19. (금) 13:53:36 KST :  12151 / 4194304
2017. 05. 19. (금) 13:53:39 KST :  19715 / 4194304
2017. 05. 19. (금) 13:53:40 KST :  21730 / 4194304
2017. 05. 19. (금) 13:53:48 KST :  37222 / 4194304
2017. 05. 19. (금) 13:53:49 KST :  38920 / 4194304
2017. 05. 19. (금) 13:53:50 KST :  40596 / 4194304
2017. 05. 19. (금) 13:53:53 KST :  45767 / 4194304
2017. 05. 19. (금) 13:53:55 KST :  47409 / 4194304
2017. 05. 19. (금) 13:53:57 KST :  50770 / 4194304
2017. 05. 19. (금) 13:53:58 KST :  52357 / 4194304
2017. 05. 19. (금) 13:53:59 KST :  53992 / 4194304
2017. 05. 19. (금) 13:54:00 KST :  55560 / 4194304
2017. 05. 19. (금) 13:54:01 KST :  57140 / 4194304
2017. 05. 19. (금) 13:54:02 KST :  58643 / 4194304
2017. 05. 19. (금) 13:54:03 KST :  60146 / 4194304
2017. 05. 19. (금) 13:54:08 KST :  55721 / 4194304
2017. 05. 19. (금) 13:54:09 KST :  54887 / 4194304
2017. 05. 19. (금) 13:54:10 KST :  54108 / 4194304
2017. 05. 19. (금) 13:54:11 KST :  53541 / 4194304
2017. 05. 19. (금) 13:54:12 KST :  53018 / 4194304
2017. 05. 19. (금) 13:54:15 KST :  51364 / 4194304
2017. 05. 19. (금) 13:54:21 KST :  49690 / 4194304
2017. 05. 19. (금) 13:54:27 KST :  48690 / 4194304
2017. 05. 19. (금) 13:54:30 KST :  48029 / 4194304
```

위 포맷은 date : conntract_count / conntrack_max 이다.

최대 가능개수 4M개(4,194,304)인데 60k개 정도에서 최고점을 찍고 하강한다. 실제로 그 시점 쯤에 health check가 실패한다고 한다.

conntrack table의 최대 가능 개수와 무관하게 SNAT의 정보 저장에 한계가 있다.

생각해 보면 당연하다.
SNAT의 bucket entry는 (소스 IP, 소스 포트, 목적 IP, 목적 포트)와 같이
unique tuple들로 정의된다.
이 4개의 element가 하나라도 다르면 다른 entry가 된다.

예) unique한 SNAT entry

    1.1.1.1:50135 -> 10.100.13.3:53
    1.1.1.1:50136 -> 10.100.13.3:53
    1.2.1.1:50135 -> 10.100.13.3:53
    ...

kube-proxy iptables의 문제는 incoming traffic을 모두 SNAT하기 때문에
소스 IP가 같다는 것이다.
그리고 bind container는 한 개이기 때문에 목적 IP와 목적 port도 동일하다.
그렇다면, unique하기 위해서는 소스 포트를 다르게 할 수 밖에 없다.

예)

    1.1.1.1:50135 -> 10.100.13.3:53
    1.1.1.1:50136 -> 10.100.13.3:53
    1.1.1.1:50137 -> 10.100.13.3:53


여기서 포트에 대해 잠시 고찰해 보자.

TCP/UDP 포트는 16bit unsigned로 되어 0 ~ 65535 범위에서 할당할 수 있다.

전체 할당 가능한 port 수가 이론적으로는 65536개이나 이미 사용 중인 포트를 제외
하면, ephemeral port의 개수는 60k 정도가 될 것이다.

kube-proxy iptables가 SNAT을 하면, 60k 정도의 dns 세션 처리가 가능하다. 만약 그 이상의 query가 들어오면, SNAT 소스 포트 할당을 할 수 없게 된다.

결국 이 SNAT이 kubernetes 성능 저하의 원인이 된다.

## 질문과 대답

Q. 이전에 4개의 container를 띄워 분산하면 1개가 처리하는 것보다 rps가 더 잘
나왔는데, SNAT이 문제라면 이것은 어떻게 설명할 수 있는가?

A. 목적 IP가 분산되기 때문이다.
SNAT의 기본 unique 단위는 (소스 IP, 소스 포트, 목적 IP, 목적 포트) tuple이다.

예)

    1.1.1.1:50135 -> 10.100.13.3:53
    1.1.1.1:50136 -> 10.100.13.3:53
    1.1.1.1:50135 -> 10.100.13.4:53
    1.1.1.1:50136 -> 10.100.13.4:53

그러므로 container 개수를 늘리면 어느 정도 성능 향상이 된다.

실제로 container 개수를 늘려 가면서 성능 시험을 하기도 했다.
20개까지 늘려 보았으나 10,000 - 12,000 rps정도에서 수렴한다.
그 이상 container를 늘려도 시스템 과부하로 성능이 나오지 않는다.

과연 한 물리 머신에 20개의 container를 띄워 서비스하는 것이 의미가 있을까?


## 결론

* kubernetes의 external IP 서비스 기능으로 부하 부산을 하는 방식으로는
SNAT으로 인한 성능 저하때문에  고성능의 서비스를 하기에 어렵다.
* 고성능 서비스를 위해서는 외부 Load Balancer를 사용하는 것이 좋다.

## 에필로그 : health check의 기준 문제

위에서 말한대로 health check는 100개의 query를 던져 100개 다 응답을 받으면 
healthy하다고 간주한다. 
의문점은 100개를 다 받는데 기다리는 시간(response wait time, RWT) 설정이 
어떻게 되어 있는 가 하는 것이다.

고객에게 문의하였으나 이 설정값을 모른다고 한다.
health check program이 Perl 로 작성된 것이라니 찾는 것은 그리 어렵지 않다.
이 RWT가 너무 짧다면 health check의 기준이 너무 높게 설정된 것이 아닌가 
하는 것이다.
예를 들어, RWT를 1초로 했다면, bind container가 1.1초 후에 응답했더라도 실패가 된다는 것이다.
성능 실험을 할 때 이런 기준점은 알고 하는 것이 좋다.


## Appendix

* kube-proxy iptables (참고: 외부 IP 정보는 x.x.x.x로 변환함)
```
*nat
:PREROUTING ACCEPT [2754:238511]
:INPUT ACCEPT [3:195]
:OUTPUT ACCEPT [6:360]
:POSTROUTING ACCEPT [6:360]
:DOCKER - [0:0]
:KUBE-MARK-DROP - [0:0]
:KUBE-MARK-MASQ - [0:0]
:KUBE-NODEPORTS - [0:0]
:KUBE-POSTROUTING - [0:0]
:KUBE-SEP-4HNHNBOBUT2EV3H2 - [0:0]
:KUBE-SEP-BCSN2HMZQJBRJOX4 - [0:0]
:KUBE-SEP-UPD7ND2TPNSDBE6L - [0:0]
:KUBE-SERVICES - [0:0]
:KUBE-SVC-357LDXEIX7XFORLA - [0:0]
:KUBE-SVC-NPX46M4PTMTKRN6Y - [0:0]
:KUBE-SVC-QVE7WEGHBCUIDG2Z - [0:0]
-A PREROUTING -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
-A PREROUTING -m addrtype --dst-type LOCAL -j DOCKER
-A OUTPUT -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
-A OUTPUT ! -d 127.0.0.0/8 -m addrtype --dst-type LOCAL -j DOCKER
-A POSTROUTING -s 10.100.13.0/24 ! -o docker0 -j MASQUERADE
-A POSTROUTING -m comment --comment "kubernetes postrouting rules" -j KUBE-POSTROUTING
-A DOCKER -i docker0 -j RETURN
-A KUBE-MARK-DROP -j MARK --set-xmark 0x8000/0x8000
-A KUBE-MARK-MASQ -j MARK --set-xmark 0x4000/0x4000
-A KUBE-POSTROUTING -m comment --comment "kubernetes service traffic requiring SNAT" -m mark --mark 0x4000/0x4000 -j MASQUERADE
-A KUBE-SEP-4HNHNBOBUT2EV3H2 -s 10.100.13.3/32 -m comment --comment "default/ares-k29-bindapp:dns-tcp-53" -j KUBE-MARK-MASQ
-A KUBE-SEP-4HNHNBOBUT2EV3H2 -p tcp -m comment --comment "default/ares-k29-bindapp:dns-tcp-53" -m tcp -j DNAT --to-destination 10.100.13.3:53
-A KUBE-SEP-BCSN2HMZQJBRJOX4 -s 10.100.13.3/32 -m comment --comment "default/ares-k29-bindapp:dns-udp-53" -j KUBE-MARK-MASQ
-A KUBE-SEP-BCSN2HMZQJBRJOX4 -p udp -m comment --comment "default/ares-k29-bindapp:dns-udp-53" -m udp -j DNAT --to-destination 10.100.13.3:53
-A KUBE-SERVICES ! -s 10.24.0.0/16 -d 10.24.0.1/32 -p tcp -m comment --comment "default/kubernetes:https cluster IP" -m tcp --dport 443 -j KUBE-MARK-MASQ
-A KUBE-SERVICES -d 10.24.0.1/32 -p tcp -m comment --comment "default/kubernetes:https cluster IP" -m tcp --dport 443 -j KUBE-SVC-NPX46M4PTMTKRN6Y
-A KUBE-SERVICES ! -s 10.24.0.0/16 -d 10.24.138.29/32 -p udp -m comment --comment "default/ares-k29-bindapp:dns-udp-53 cluster IP" -m udp --dport 53 -j KUBE-MARK-MASQ
-A KUBE-SERVICES -d 10.24.138.29/32 -p udp -m comment --comment "default/ares-k29-bindapp:dns-udp-53 cluster IP" -m udp --dport 53 -j KUBE-SVC-357LDXEIX7XFORLA
-A KUBE-SERVICES -d 10.5.6.29/32 -p udp -m comment --comment "default/ares-k29-bindapp:dns-udp-53 external IP" -m udp --dport 53 -j KUBE-MARK-MASQ
-A KUBE-SERVICES -d 10.5.6.29/32 -p udp -m comment --comment "default/ares-k29-bindapp:dns-udp-53 external IP" -m udp --dport 53 -m physdev ! --physdev-is-in -m addrtype ! --src-type LOCAL -j KUBE-SVC-357LDXEIX7XFORLA
-A KUBE-SERVICES -d 10.5.6.29/32 -p udp -m comment --comment "default/ares-k29-bindapp:dns-udp-53 external IP" -m udp --dport 53 -m addrtype --dst-type LOCAL -j KUBE-SVC-357LDXEIX7XFORLA
-A KUBE-SERVICES -d x.x.x.x/32 -p udp -m comment --comment "default/ares-k29-bindapp:dns-udp-53 external IP" -m udp --dport 53 -j KUBE-MARK-MASQ
-A KUBE-SERVICES -d x.x.x.x/32 -p udp -m comment --comment "default/ares-k29-bindapp:dns-udp-53 external IP" -m udp --dport 53 -m physdev ! --physdev-is-in -m addrtype ! --src-type LOCAL -j KUBE-SVC-357LDXEIX7XFORLA
-A KUBE-SERVICES -d x.x.x.x/32 -p udp -m comment --comment "default/ares-k29-bindapp:dns-udp-53 external IP" -m udp --dport 53 -m addrtype --dst-type LOCAL -j KUBE-SVC-357LDXEIX7XFORLA
-A KUBE-SERVICES ! -s 10.24.0.0/16 -d 10.24.138.29/32 -p tcp -m comment --comment "default/ares-k29-bindapp:dns-tcp-53 cluster IP" -m tcp --dport 53 -j KUBE-MARK-MASQ
-A KUBE-SERVICES -d 10.24.138.29/32 -p tcp -m comment --comment "default/ares-k29-bindapp:dns-tcp-53 cluster IP" -m tcp --dport 53 -j KUBE-SVC-QVE7WEGHBCUIDG2Z
-A KUBE-SERVICES -d 10.5.6.29/32 -p tcp -m comment --comment "default/ares-k29-bindapp:dns-tcp-53 external IP" -m tcp --dport 53 -j KUBE-MARK-MASQ
-A KUBE-SERVICES -d 10.5.6.29/32 -p tcp -m comment --comment "default/ares-k29-bindapp:dns-tcp-53 external IP" -m tcp --dport 53 -m physdev ! --physdev-is-in -m addrtype ! --src-type LOCAL -j KUBE-SVC-QVE7WEGHBCUIDG2Z
-A KUBE-SERVICES -d 10.5.6.29/32 -p tcp -m comment --comment "default/ares-k29-bindapp:dns-tcp-53 external IP" -m tcp --dport 53 -m addrtype --dst-type LOCAL -j KUBE-SVC-QVE7WEGHBCUIDG2Z
-A KUBE-SERVICES -d x.x.x.x/32 -p tcp -m comment --comment "default/ares-k29-bindapp:dns-tcp-53 external IP" -m tcp --dport 53 -j KUBE-MARK-MASQ
-A KUBE-SERVICES -d x.x.x.x/32 -p tcp -m comment --comment "default/ares-k29-bindapp:dns-tcp-53 external IP" -m tcp --dport 53 -m physdev ! --physdev-is-in -m addrtype ! --src-type LOCAL -j KUBE-SVC-QVE7WEGHBCUIDG2Z
-A KUBE-SERVICES -d x.x.x.x/32 -p tcp -m comment --comment "default/ares-k29-bindapp:dns-tcp-53 external IP" -m tcp --dport 53 -m addrtype --dst-type LOCAL -j KUBE-SVC-QVE7WEGHBCUIDG2Z
-A KUBE-SERVICES -m comment --comment "kubernetes service nodeports; NOTE: this must be the last rule in this chain" -m addrtype --dst-type LOCAL -j KUBE-NODEPORTS
-A KUBE-SVC-357LDXEIX7XFORLA -m comment --comment "default/ares-k29-bindapp:dns-udp-53" -j KUBE-SEP-BCSN2HMZQJBRJOX4
-A KUBE-SVC-NPX46M4PTMTKRN6Y -m comment --comment "default/kubernetes:https" -m recent --rcheck --seconds 10800 --reap --name KUBE-SEP-UPD7ND2TPNSDBE6L --mask 255.255.255.255 --rsource -j KUBE-SEP-UPD7ND2TPNSDBE6L
-A KUBE-SVC-NPX46M4PTMTKRN6Y -m comment --comment "default/kubernetes:https" -j KUBE-SEP-UPD7ND2TPNSDBE6L
-A KUBE-SVC-QVE7WEGHBCUIDG2Z -m comment --comment "default/ares-k29-bindapp:dns-tcp-53" -j KUBE-SEP-4HNHNBOBUT2EV3H2
COMMIT
# Completed on Wed May 17 17:08:09 2017
# Generated by iptables-save v1.4.21 on Wed May 17 17:08:09 2017
*filter
:INPUT ACCEPT [186:70129]
:FORWARD ACCEPT [15052:982087]
:OUTPUT ACCEPT [177:15601]
:DOCKER - [0:0]
:DOCKER-ISOLATION - [0:0]
:KUBE-FIREWALL - [0:0]
:KUBE-SERVICES - [0:0]
-A INPUT -j KUBE-FIREWALL
-A FORWARD -j DOCKER-ISOLATION
-A FORWARD -o docker0 -j DOCKER
-A FORWARD -o docker0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -i docker0 ! -o docker0 -j ACCEPT
-A FORWARD -i docker0 -o docker0 -j ACCEPT
-A OUTPUT -j KUBE-FIREWALL
-A OUTPUT -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
-A DOCKER-ISOLATION -j RETURN
-A KUBE-FIREWALL -m comment --comment "kubernetes firewall for dropping marked packets" -m mark --mark 0x8000/0x8000 -j DROP
COMMIT
```

* manual DNAT iptables
```
*nat
:PREROUTING ACCEPT [2754:238511]
-A PREROUTING -i eth3 -p udp -m udp --dport 53 -m state --state NEW -j DNAT --to-destination 10.100.13.3:53
-A PREROUTING -i eth3:1 -p udp -m udp --dport 53 -m state --state NEW -j DNAT --to-destination 10.100.13.3:53
:INPUT ACCEPT [3:195]
:OUTPUT ACCEPT [6:360]
:POSTROUTING ACCEPT [6:360]
-A POSTROUTING -s 10.100.13.0/24 ! -o docker0 -j MASQUERADE
:DOCKER - [0:0]
:KUBE-MARK-DROP - [0:0]
:KUBE-MARK-MASQ - [0:0]
:KUBE-NODEPORTS - [0:0]
:KUBE-POSTROUTING - [0:0]
:KUBE-SEP-4HNHNBOBUT2EV3H2 - [0:0]
:KUBE-SEP-BCSN2HMZQJBRJOX4 - [0:0]
:KUBE-SEP-UPD7ND2TPNSDBE6L - [0:0]
:KUBE-SERVICES - [0:0]
:KUBE-SVC-357LDXEIX7XFORLA - [0:0]
:KUBE-SVC-NPX46M4PTMTKRN6Y - [0:0]
:KUBE-SVC-QVE7WEGHBCUIDG2Z - [0:0]
COMMIT
# Completed on Wed May 17 17:08:09 2017
# Generated by iptables-save v1.4.21 on Wed May 17 17:08:09 2017
*filter
:INPUT ACCEPT [186:70129]
:FORWARD ACCEPT [15052:982087]
:OUTPUT ACCEPT [177:15601]
:DOCKER - [0:0]
:DOCKER-ISOLATION - [0:0]
:KUBE-FIREWALL - [0:0]
:KUBE-SERVICES - [0:0]
COMMIT
```

## References

* man iptables
* man iptables-extensions
* https://www.digitalocean.com/community/tutorials/a-deep-dive-into-iptables-and-netfilter-architecture

