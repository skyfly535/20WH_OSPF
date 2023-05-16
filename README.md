# Статическая и динамическая маршрутизация, OSPF.

Сценарии OSPF:

1. Развернуть 3 виртуальные машины;

2. Объединить их разными vlan;

- настроить OSPF между машинами на базе Quagga;

- изобразить ассиметричный роутинг;

- сделать один из линков "дорогим", но что бы при этом роутинг был симметричным.

Цель:

Создать домашнюю сетевую лабораторию. Научится настраивать протокол OSPF в Linux-based системах.

## Развертывание стенда для демострации статической и динамической маршрутизации, OSPF.

Стэнд состоит из хостовой машины под управлением ОС Ubuntu 20.04 на которой развернут Ansible и трех виртуальных машин `ubuntu/focal64`.

Машины с именами: `router1`, `router2`, `router3` выполняют роль `роутеров`.

![Снимок экрана от 2023-05-15 23-16-18](https://github.com/skyfly535/20WH_OSPF/assets/114483769/c9d8f45e-50ad-4c79-ba07-684fb5c73c60)

схема коммутации ВМ

Разворачиваем инфраструктуру в Vagrant исключительно через Ansible.

Для развертывания стенда в каталог с Vagrantfile и Playbook необходимо поместить конфигурационный файл `ansible.cfg` и Файл инвентаризации `hosts`. 

Для каждой ВМ я создал отдельный `frr.conf` (frr1.conf, frr2.conf, frr3.conf соответственно) и привел всоответствие Playbook.

Все коментарии по каждому блоку указаны в тексте Playbook - `ospf.yml`.

Выполняем установку стенда

```
vagrant up
```

## Результат работы

### Настройка OSPF между машинами на базе Quagga

Проверям статус OSPF

```
root@router3:~# systemctl status frr
● frr.service - FRRouting
     Loaded: loaded (/lib/systemd/system/frr.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2023-05-15 11:38:29 UTC; 1h 53min ago
       Docs: https://frrouting.readthedocs.io/en/latest/setup.html
    Process: 5617 ExecStart=/usr/lib/frr/frrinit.sh start (code=exited, status=0/SUCCESS)
   Main PID: 5635 (watchfrr)
     Status: "FRR Operational"
      Tasks: 9 (limit: 1131)
     Memory: 13.7M
     CGroup: /system.slice/frr.service
             ├─5635 /usr/lib/frr/watchfrr -d -F traditional zebra ospfd staticd
             ├─5648 /usr/lib/frr/zebra -d -F traditional -A 127.0.0.1 -s 90000000
             ├─5653 /usr/lib/frr/ospfd -d -F traditional -A 127.0.0.1
             └─5656 /usr/lib/frr/staticd -d -F traditional -A 127.0.0.1

May 15 11:38:29 router3 watchfrr[5635]: [QDG3Y-BY5TN] ospfd state -> up : connect succeeded
May 15 11:38:29 router3 frrinit.sh[5617]:  * Started watchfrr
May 15 11:38:29 router3 watchfrr[5635]: [QDG3Y-BY5TN] zebra state -> up : connect succeeded
May 15 11:38:29 router3 watchfrr[5635]: [QDG3Y-BY5TN] staticd state -> up : connect succeeded
May 15 11:38:29 router3 watchfrr[5635]: [KWE5Q-QNGFC] all daemons up, doing startup-complete notify
May 15 11:38:29 router3 systemd[1]: Started FRRouting.
May 15 11:38:59 router3 ospfd[5653]: [S5PCG-77H23] Packet[DD]: Neighbor 1.1.1.1 Negotiation done (Master).
May 15 11:38:59 router3 ospfd[5653]: [S5PCG-77H23] Packet[DD]: Neighbor 2.2.2.2 Negotiation done (Master).
```
Проверим доступность "внешних" сетей с хоста `router3` на `router1` и `router2` соответственно.

```
root@router3:~# ping 192.168.10.1
PING 192.168.10.1 (192.168.10.1) 56(84) bytes of data.
64 bytes from 192.168.10.1: icmp_seq=1 ttl=64 time=1.04 ms
64 bytes from 192.168.10.1: icmp_seq=2 ttl=64 time=1.25 ms
64 bytes from 192.168.10.1: icmp_seq=3 ttl=64 time=1.14 ms
^C
--- 192.168.10.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2050ms
rtt min/avg/max/mdev = 1.044/1.142/1.246/0.082 ms
root@router3:~# ping 192.168.20.1
PING 192.168.20.1 (192.168.20.1) 56(84) bytes of data.
64 bytes from 192.168.20.1: icmp_seq=1 ttl=64 time=0.431 ms
64 bytes from 192.168.20.1: icmp_seq=2 ttl=64 time=1.29 ms
64 bytes from 192.168.20.1: icmp_seq=3 ttl=64 time=1.05 ms
^C
--- 192.168.20.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2029ms
rtt min/avg/max/mdev = 0.431/0.924/1.291/0.362 ms
```

Запустим трассировку до адреса 192.168.10.1

```
root@router3:~# traceroute 192.168.10.1
traceroute to 192.168.10.1 (192.168.10.1), 30 hops max, 60 byte packets
 1  192.168.10.1 (192.168.10.1)  1.117 ms  0.957 ms  0.471 ms
```

Отключим интерфейс enp0s9 и немного подождем и снова запустим трассировку до ip-адреса 192.168.10.1

```
root@router3:~# ifconfig enp0s9 down
root@router3:~# ip a | grep enp0s9
4: enp0s9: <BROADCAST,MULTICAST> mtu 1500 qdisc fq_codel state DOWN group default qlen 1000
root@router3:~# traceroute 192.168.10.1
traceroute to 192.168.10.1 (192.168.10.1), 30 hops max, 60 byte packets
 1  10.0.11.2 (10.0.11.2)  0.939 ms  1.273 ms  1.174 ms
 2  192.168.10.1 (192.168.10.1)  2.148 ms  2.726 ms  2.514 ms
```
Как мы видим, после отключения интерфейса пакеты достигли интефейса 192.168.10.0/24 по альтернативному маршруту.

### Настройка ассиметричного роутинга.

Отработку ассиметричного роутинга я выполнял на интерфейсе `enp0s8` `router3`

Для разворачивания стенда с ассиметричным роутингом меняем в начальном Playbook - `ospf.yml` одну `task`

```
 # Отключаем запрет ассиметричного роутинга 
  - name: set up asynchronous routing
    sysctl:
      name: net.ipv4.conf.all.rp_filter
      value: '0'
      state: present
```
и в файле `frr3.conf` в блоке интерфейса `enp0s8` раскоментируем строку 

```
!ip ospf cost 1000 ---> ip ospf cost 1000
```
Проверяем `router3`

```
router3# show ip route ospf
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

O>* 10.0.10.0/30 [110/200] via 10.0.12.1, enp0s9, weight 1, 00:00:26
O   10.0.11.0/30 [110/300] via 10.0.12.1, enp0s9, weight 1, 00:00:26
O   10.0.12.0/30 [110/100] is directly connected, enp0s9, weight 1, 00:14:58
O>* 192.168.10.0/24 [110/200] via 10.0.12.1, enp0s9, weight 1, 00:14:58
O>* 192.168.20.0/24 [110/300] via 10.0.12.1, enp0s9, weight 1, 00:00:26
O   192.168.30.0/24 [110/100] is directly connected, enp0s10, weight 1, 01:54:54
```
Проверяем `router2`

```
router2# show ip route ospf
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

O   10.0.10.0/30 [110/100] is directly connected, enp0s8, weight 1, 02:07:44
O   10.0.11.0/30 [110/100] is directly connected, enp0s9, weight 1, 02:07:08
O>* 10.0.12.0/30 [110/200] via 10.0.10.1, enp0s8, weight 1, 00:27:48
  *                        via 10.0.11.1, enp0s9, weight 1, 00:27:48
O>* 192.168.10.0/24 [110/200] via 10.0.10.1, enp0s8, weight 1, 02:06:58
O   192.168.20.0/24 [110/100] is directly connected, enp0s10, weight 1, 02:07:43
O>* 192.168.30.0/24 [110/200] via 10.0.11.1, enp0s9, weight 1, 02:06:59
```
После внесения данных настроек, мы видим, что прямой и обратный маршруты до сети 192.168.20.0/30  через разные сетевые интерфейсы:

На `router3` запускаем пинг от 192.168.30.1 до 192.168.20.1:

```
 root@router3:~# ping -I 192.168.30.1 192.168.20.1
PING 192.168.20.1 (192.168.20.1) from 192.168.30.1 : 56(84) bytes of data.
64 bytes from 192.168.20.1: icmp_seq=1 ttl=64 time=0.638 ms
64 bytes from 192.168.20.1: icmp_seq=2 ttl=64 time=1.55 ms
64 bytes from 192.168.20.1: icmp_seq=3 ttl=64 time=1.45 ms
64 bytes from 192.168.20.1: icmp_seq=4 ttl=64 time=2.00 ms
64 bytes from 192.168.20.1: icmp_seq=5 ttl=64 time=0.538 ms
64 bytes from 192.168.20.1: icmp_seq=6 ttl=64 time=1.60 ms
```

Запускаем tcpdump на интерфейсе `enp0s9` `router2`:

```
root@router2:/home/vagrant# tcpdump -i enp0s9
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp0s9, link-type EN10MB (Ethernet), capture size 262144 bytes
10:16:48.645524 IP router2 > 192.168.30.1: ICMP echo reply, id 6, seq 31, length 64
10:16:49.189433 IP router2 > ospf-all.mcast.net: OSPFv2, Hello, length 48
10:16:49.191090 IP 10.0.12.2 > ospf-all.mcast.net: OSPFv2, Hello, length 44
10:16:49.191090 IP 10.0.11.1 > ospf-all.mcast.net: OSPFv2, Hello, length 48
10:16:49.646484 IP router2 > 192.168.30.1: ICMP echo reply, id 6, seq 32, length 64
10:16:50.648792 IP router2 > 192.168.30.1: ICMP echo reply, id 6, seq 33, length 64
10:16:51.649479 IP router2 > 192.168.30.1: ICMP echo reply, id 6, seq 34, length 64
10:16:52.650684 IP router2 > 192.168.30.1: ICMP echo reply, id 6, seq 35, length 64
10:16:53.652699 IP router2 > 192.168.30.1: ICMP echo reply, id 6, seq 36, length 64
10:16:54.654200 IP router2 > 192.168.30.1: ICMP echo reply, id 6, seq 37, length 64
10:16:55.696201 IP router2 > 192.168.30.1: ICMP echo reply, id 6, seq 38, length 64
10:16:56.698359 IP router2 > 192.168.30.1: ICMP echo reply, id 6, seq 39, length 64
10:16:57.699661 IP router2 > 192.168.30.1: ICMP echo reply, id 6, seq 40, length 64
10:16:58.717327 IP router2 > 192.168.30.1: ICMP echo reply, id 6, seq 41, length 64
10:16:59.189718 IP router2 > ospf-all.mcast.net: OSPFv2, Hello, length 48
10:16:59.191165 IP 10.0.12.2 > ospf-all.mcast.net: OSPFv2, Hello, length 44
10:16:59.191534 IP 10.0.11.1 > ospf-all.mcast.net: OSPFv2, Hello, length 48
```

Запускаем tcpdump на интерфейсе `enp0s8` `router2`:

```
root@router2:/home/vagrant# tcpdump -i enp0s8
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp0s8, link-type EN10MB (Ethernet), capture size 262144 bytes
10:17:18.804425 IP 192.168.30.1 > router2: ICMP echo request, id 6, seq 61, length 64
10:17:19.189866 IP router2 > ospf-all.mcast.net: OSPFv2, Hello, length 48
10:17:19.806173 IP 192.168.30.1 > router2: ICMP echo request, id 6, seq 62, length 64
10:17:20.818461 IP 192.168.30.1 > router2: ICMP echo request, id 6, seq 63, length 64
10:17:21.820451 IP 192.168.30.1 > router2: ICMP echo request, id 6, seq 64, length 64
10:17:22.821872 IP 192.168.30.1 > router2: ICMP echo request, id 6, seq 65, length 64
10:17:23.823107 IP 192.168.30.1 > router2: ICMP echo request, id 6, seq 66, length 64
10:17:24.823877 IP 192.168.30.1 > router2: ICMP echo request, id 6, seq 67, length 64
10:17:25.528233 IP 10.0.10.1 > ospf-all.mcast.net: OSPFv2, Hello, length 48
10:17:25.825398 IP 192.168.30.1 > router2: ICMP echo request, id 6, seq 68, length 64
10:17:26.826376 IP 192.168.30.1 > router2: ICMP echo request, id 6, seq 69, length 64
10:17:27.069595 ARP, Request who-has router2 tell 10.0.10.1, length 46
10:17:27.069665 ARP, Reply router2 is-at 08:00:27:e1:77:1d (oui Unknown), length 28
10:17:27.828268 IP 192.168.30.1 > router2: ICMP echo request, id 6, seq 70, length 64
10:17:28.829898 IP 192.168.30.1 > router2: ICMP echo request, id 6, seq 71, length 64
10:17:29.190858 IP router2 > ospf-all.mcast.net: OSPFv2, Hello, length 48
10:17:29.833103 IP 192.168.30.1 > router2: ICMP echo request, id 6, seq 72, length 64
10:17:30.834455 IP 192.168.30.1 > router2: ICMP echo request, id 6, seq 73, length 64
10:17:31.835441 IP 192.168.30.1 > router2: ICMP echo request, id 6, seq 74, length 64
10:17:32.858142 IP 192.168.30.1 > router2: ICMP echo request, id 6, seq 75, length 64
10:17:33.858752 IP 192.168.30.1 > router2: ICMP echo request, id 6, seq 76, length 64
10:17:34.859487 IP 192.168.30.1 > router2: ICMP echo request, id 6, seq 77, length 64
10:17:35.529616 IP 10.0.10.1 > ospf-all.mcast.net: OSPFv2, Hello, length 48
```
видим что прямые и обратные интерфейсы используют разные интерфейсы.

### Настройка симметичного роутинга.

У нас уже есть один «дорогой» интерфейс, нам потребуется добавить ещё один дорогой интерфейс, чтобы у нас перестала работать ассиметричная маршрутизация. 

Так как в прошлом задании мы заметили что router2 будет отправлять обратно трафик через порт enp0s8, мы также должны сделать его дорогим и далее проверить, что теперь используется симметричная маршрутизация:

Делаем интерфейс `enp0s9` на `router2` дорогим, тем самым балансируем систему. 

Для разворачивания стенда с симетричным роутингом меняем в файле `frr2.conf` в блоке интерфейса `enp0s9` раскоментируем строку 

```
!ip ospf cost 1000 ---> ip ospf cost 1000
```

Снова на `router3` запускаем пинг от 192.168.30.1 до 192.168.20.1:

```
 root@router3:~# ping -I 192.168.30.1 192.168.20.1
PING 192.168.20.1 (192.168.20.1) from 192.168.30.1 : 56(84) bytes of data.
64 bytes from 192.168.20.1: icmp_seq=1 ttl=64 time=1.39 ms
64 bytes from 192.168.20.1: icmp_seq=2 ttl=64 time=1.07 ms
64 bytes from 192.168.20.1: icmp_seq=3 ttl=64 time=0.329 ms
64 bytes from 192.168.20.1: icmp_seq=4 ttl=64 time=1.06 ms
64 bytes from 192.168.20.1: icmp_seq=5 ttl=64 time=1.16 ms
64 bytes from 192.168.20.1: icmp_seq=6 ttl=64 time=1.72 ms
64 bytes from 192.168.20.1: icmp_seq=7 ttl=64 time=1.20 ms
64 bytes from 192.168.20.1: icmp_seq=8 ttl=64 time=1.44 ms
64 bytes from 192.168.20.1: icmp_seq=9 ttl=64 time=1.10 ms
64 bytes from 192.168.20.1: icmp_seq=10 ttl=64 time=1.05 ms
64 bytes from 192.168.20.1: icmp_seq=11 ttl=64 time=1.12 ms
```

Запускаем tcpdump на интерфейсе `enp0s9` `router2`:

```
root@router2:/home/vagrant# tcpdump -i enp0s9
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp0s9, link-type EN10MB (Ethernet), capture size 262144 bytes
11:33:12.866453 IP 192.168.30.1 > router2: ICMP echo request, id 8, seq 31, length 64
11:33:12.866573 IP router2 > 192.168.30.1: ICMP echo reply, id 8, seq 31, length 64
11:33:13.867794 IP 192.168.30.1 > router2: ICMP echo request, id 8, seq 32, length 64
11:33:13.867871 IP router2 > 192.168.30.1: ICMP echo reply, id 8, seq 32, length 64
11:33:14.827778 IP 10.0.12.2 > ospf-all.mcast.net: OSPFv2, Hello, length 44
11:33:14.827780 IP 10.0.11.1 > ospf-all.mcast.net: OSPFv2, Hello, length 48
11:33:14.868509 IP 192.168.30.1 > router2: ICMP echo request, id 8, seq 33, length 64
11:33:14.868580 IP router2 > 192.168.30.1: ICMP echo reply, id 8, seq 33, length 64
11:33:15.870256 IP 192.168.30.1 > router2: ICMP echo request, id 8, seq 34, length 64
11:33:15.870320 IP router2 > 192.168.30.1: ICMP echo reply, id 8, seq 34, length 64
11:33:16.025993 IP router2 > ospf-all.mcast.net: OSPFv2, Hello, length 48
11:33:16.871922 IP 192.168.30.1 > router2: ICMP echo request, id 8, seq 35, length 64
11:33:16.872007 IP router2 > 192.168.30.1: ICMP echo reply, id 8, seq 35, length 64
11:33:17.874007 IP 192.168.30.1 > router2: ICMP echo request, id 8, seq 36, length 64
11:33:17.874071 IP router2 > 192.168.30.1: ICMP echo reply, id 8, seq 36, length 64
11:33:18.875607 IP 192.168.30.1 > router2: ICMP echo request, id 8, seq 37, length 64
11:33:18.875694 IP router2 > 192.168.30.1: ICMP echo reply, id 8, seq 37, length 64
11:33:19.877349 IP 192.168.30.1 > router2: ICMP echo request, id 8, seq 38, length 64
11:33:19.877411 IP router2 > 192.168.30.1: ICMP echo reply, id 8, seq 38, length 64
11:33:20.879029 IP 192.168.30.1 > router2: ICMP echo request, id 8, seq 39, length 64
11:33:20.879093 IP router2 > 192.168.30.1: ICMP echo reply, id 8, seq 39, length 64
```
Видим, что прямые и обратные пакеты идут через один интерфейс.
