# 01. Proxmox 설치

## 목표

물리 서버에 Proxmox VE를 설치하고 Web UI까지 접속할 수 있는 상태를 만든다.

## 설치 이미지 준비

Proxmox VE ISO를 내려받아 Rufus 등의 도구로 USB 설치 미디어를 만든다.

부팅 후 설치 방식은 일반적으로 **Graphical**을 선택하면 된다. Terminal UI를 사용해도 설치 결과 자체는 동일하다.

## 네트워크 설정

설치 과정에서 다음 항목을 설정한다.

- Management Interface: 관리 네트워크에 연결된 NIC
- Hostname (FQDN): 예) `n100-01.home.arpa`
- IP Address: 고정 IP 권장
- Gateway: 홈 라우터 주소
- DNS Server: 사용하는 DNS 서버
- CIDR: 일반적인 /24 네트워크라면 `/24`

예시:

```text
Hostname: n100-01.home.arpa
IP:       192.168.0.251/24
Gateway:  192.168.0.1
```

> 실제 IP는 자신의 네트워크 대역에 맞게 설정한다.

## FQDN 이름 규칙

홈랩에서는 다음처럼 노드 자체를 나타내는 이름을 권장한다.

```text
n100-01.home.arpa
n100-02.home.arpa
```

- `n100-01`: 물리 노드 이름
- `home.arpa`: 홈 네트워크용 도메인

VM 이름에 물리 호스트 이름을 넣을 필요는 없다. VM은 나중에 다른 노드로 이동할 수 있기 때문이다.

## 첫 로그인

설치가 끝나면 로컬 콘솔은 CLI가 정상이다. 관리 UI는 브라우저에서 접속한다.

```text
https://<PROXMOX-IP>:8006
```

로그인:

```text
User:  root
Realm: Linux PAM standard authentication
```

자체 서명 인증서를 사용하므로 처음 접속할 때 브라우저 인증서 경고가 나타날 수 있다.

## 구독 메시지

Proxmox VE 자체는 무료로 사용할 수 있다. Web UI의 "유효한 구독이 없습니다" 메시지는 유료 서포트/엔터프라이즈 저장소 구독 여부에 대한 안내다.

다음 장: [호스트명과 네트워크](02-hostname-network.md)
