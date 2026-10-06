# 03. 2노드 클러스터 구성

## 목표

`n100-01`, `n100-02`를 하나의 Proxmox Cluster로 묶는다.

클러스터를 구성하면 한 Web UI에서 두 노드를 함께 관리할 수 있다.

## 사전 조건

- 두 노드의 hostname이 서로 다를 것
- 각 노드가 서로의 관리 IP에 안정적으로 접근 가능할 것
- 시간 동기화가 정상일 것
- 이미 운영 중인 VM이 많은 노드를 뒤늦게 합치는 것보다 초기 구축 시점에 구성하는 것이 편하다

## 첫 번째 노드에서 Cluster 생성

`n100-01`에서:

```text
Datacenter
→ Cluster
→ Create Cluster
```

Cluster Name을 지정한다.

Cluster Network는 두 노드가 서로 통신할 관리 네트워크 인터페이스를 선택한다. 별도의 Corosync 전용망이 없다면 홈랩에서는 관리망을 사용하는 구성이 흔하다.

## 두 번째 노드 Join

`n100-01`의 Cluster Join Information을 확인한 뒤 `n100-02`에서:

```text
Datacenter
→ Cluster
→ Join Cluster
```

Join Information과 필요한 인증 정보를 입력한다.

완료되면 Datacenter 아래에 두 노드가 함께 보인다.

## Link 항목

Cluster 생성 시 보이는 Link는 Corosync 통신 경로다.

복수의 독립적인 네트워크 경로를 구성하지 않았다면 일반적으로 Link 0 하나만 사용하면 된다.

## Web UI 접근

클러스터에 Join했다고 해서 두 번째 노드 Web UI가 사라지는 것은 아니다.

각 노드의 주소로 계속 접속 가능하다.

```text
https://n100-01-IP:8006
https://n100-02-IP:8006
```

어느 노드로 접속해도 클러스터 전체 리소스를 볼 수 있다.

## 2노드 클러스터의 주의점

2노드만으로 구성하면 한 노드 장애 시 **quorum** 문제가 생길 수 있다.

즉 2노드 클러스터는 관리 편의성은 제공하지만, 이것만으로 완전한 HA 구성이 되는 것은 아니다.

HA가 목적이라면 일반적으로 다음을 별도로 검토한다.

- 3번째 voting member
- QDevice
- 공유/분산 스토리지
- fencing 및 장애 복구 정책

따라서 이 홈랩의 초기 단계에서는 VM 생성 화면의 `Add to HA`를 무조건 켜지 않는다.

다음 장: [스토리지 이해](04-storage.md)
