# homelab-proxmox

N100 기반 미니 PC 2대로 구성한 Proxmox 홈랩 구축 기록입니다.

이 문서는 **Proxmox 설치 → 네트워크/호스트명 → 클러스터 → 스토리지 이해 → Ubuntu VM 생성 → 스냅샷 → Cloud-Init 템플릿 생성** 순서로 정리합니다.

## 환경

- Proxmox VE
- 노드
  - `n100-01.home.arpa`
  - `n100-02.home.arpa`
- 2-node cluster
- 각 노드의 로컬 스토리지
  - `local`
  - `local-lvm`
- 게스트 OS: Ubuntu Server
- 템플릿: Ubuntu Cloud Image + Cloud-Init

> IP 주소, 디스크 크기, 메모리 크기 등은 각 홈랩 환경에 맞게 변경합니다.

## 목차

1. [Proxmox 설치](docs/01-proxmox-install.md)
2. [호스트명과 네트워크](docs/02-hostname-network.md)
3. [2노드 클러스터 구성](docs/03-cluster.md)
4. [local / local-lvm 스토리지 이해](docs/04-storage.md)
5. [Ubuntu Server VM 생성](docs/05-ubuntu-vm.md)
6. [스냅샷과 기본 운영](docs/06-snapshot-maintenance.md)
7. [Ubuntu Cloud Image 준비](docs/07-cloud-image.md)
8. [Cloud-Init 템플릿 생성](docs/08-cloud-init-template.md)
9. [템플릿 복제와 멀티 노드 주의사항](docs/09-template-clone-multinode.md)

## 추가 구축 사례

- [RTX 4080 SUPER GPU 패스스루 VM 구축 및 문제 해결](docs/gpu-passthrough/README.md)

## 기본 원칙

- Proxmox 노드 이름과 VM 이름은 역할이 다르므로 분리해서 생각한다.
- VM 이름과 게스트 OS hostname은 가능하면 동일하게 맞춘다.
- `local-lvm`은 노드별 로컬 스토리지다.
- 스냅샷은 빠른 롤백 수단이지 백업의 대체재가 아니다.
- 여러 Ubuntu VM을 만들 계획이라면 ISO 반복 설치보다 Cloud-Init 템플릿을 사용한다.
