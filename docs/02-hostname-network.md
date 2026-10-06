# 02. 호스트명과 네트워크

## 노드 이름

이 홈랩에서는 물리 서버를 다음처럼 구분한다.

```text
n100-01.home.arpa
n100-02.home.arpa
```

클러스터에 들어간 뒤에도 각 노드의 Web UI는 각 노드 IP의 `:8006`으로 접근할 수 있다.

## Proxmox 관리 IP

Proxmox는 일반적으로 `vmbr0` Linux Bridge에 관리 IP를 둔다.

개념적으로:

```text
물리 NIC
   │
 vmbr0
   ├─ Proxmox Host
   ├─ VM 100
   ├─ VM 101
   └─ LXC ...
```

VM이 `vmbr0`에 연결되면 홈 라우터 입장에서는 별도의 네트워크 장비처럼 보인다.

예:

```text
n100-01        192.168.0.251
n100-02        192.168.0.252
ubu-dev01      192.168.0.120
ubu-dev02      192.168.0.121
```

## DHCP 예약과 Proxmox 고정 IP

라우터에서 MAC 주소에 DHCP 예약을 걸어두더라도 Proxmox 자체에 다른 정적 IP를 설정하면 Proxmox는 그 정적 IP를 사용한다.

같은 IP를 다른 장비가 사용하지 않도록 라우터의 DHCP 풀/예약과 Proxmox 정적 IP 구성을 일관되게 관리한다.

## VM 네트워크

Ubuntu VM 생성 시 일반적인 설정:

```text
Bridge: vmbr0
Model:  VirtIO
```

Ubuntu 내부에서는 DHCP로 시작한 뒤 필요하면 라우터 DHCP 예약 또는 Ubuntu 정적 IP로 고정한다.

## DNS

라우터가 제공하는 DNS를 사용해도 되고, 별도 공용 DNS를 사용할 수도 있다.

DNS 선택은 Proxmox 동작 자체보다 이름 해석 정책의 문제다. 내부 도메인을 사용한다면 내부 DNS가 해당 이름을 해석할 수 있도록 구성해야 한다.

다음 장: [2노드 클러스터 구성](03-cluster.md)
