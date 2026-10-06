# 09. 템플릿 복제와 멀티 노드 주의사항

## Clone

Template에서:

```text
Template 우클릭
→ Clone
```

일반적인 홈랩에서는 독립된 디스크를 갖는 **Full Clone**이 이해하기 쉽다.

예:

```text
Template: ubuntu-2404-template

Clone:
  VM ID: 101
  Name: ubu-dev02
```

## hostname

ISO 설치에서는 Ubuntu 설치 과정에서 Server name을 직접 입력한다.

Cloud-Init 방식에서는 Clone VM의 첫 부팅 시 Cloud-Init이 hostname을 설정한다.

따라서 Proxmox VM Name과 Ubuntu hostname을 같은 규칙으로 관리하기 쉽다.

첫 부팅 후 확인:

```bash
hostname
hostnamectl
cloud-init status
```

정상 완료 예:

```text
status: done
```

## local-lvm과 다른 노드

현재 구성에서 가장 중요한 제한:

```text
n100-01/local-lvm
≠
n100-02/local-lvm
```

n100-01의 `local-lvm`에 저장된 Template Disk는 n100-02가 직접 사용하는 공유 디스크가 아니다.

따라서 템플릿을 여러 노드에서 운영하는 방법은 별도로 설계해야 한다.

## 단순한 방법: 노드별 Template

2노드 홈랩에서 가장 단순한 방법:

```text
n100-01
└─ 9000 ubuntu-2404-template-n1

n100-02
└─ 9001 ubuntu-2404-template-n2
```

같은 Cloud Image와 동일한 설정으로 각 노드에 템플릿을 하나씩 만든다.

VM ID는 클러스터 전체에서 중복될 수 없으므로 서로 다른 ID를 사용한다.

## 공유 이미지 방식

노드가 늘어나거나 이미지 관리가 번거로워지면 공유 스토리지 또는 이미지 자동 배포를 고려한다.

예:

```text
Shared Storage
└─ Ubuntu Template
    ├─ Clone → n100-01
    └─ Clone → n100-02
```

선택지:

- NFS
- iSCSI
- Ceph
- 기타 Proxmox가 지원하는 공유 스토리지

2노드 N100 홈랩에서 Ceph는 리소스/쿼럼/네트워크 요구사항 때문에 초기 구성으로는 과할 수 있다.

## 클라우드에 가까운 방향

규모가 커지면 다음 조합을 고려한다.

```text
Ubuntu Cloud Image
        ↓
Packer
        ↓
표준 Template
        ↓
Terraform
        ↓
VM 생성
        ↓
Cloud-Init / Ansible
        ↓
hostname, user, SSH key, network, package 구성
```

핵심은 **공유/표준 이미지 + 자동 프로비저닝**이다.

## 현재 홈랩 추천 단계

처음부터 모든 것을 자동화하지 않는다.

1. Cloud-Init 템플릿을 이해한다.
2. 필요하면 두 노드에 동일 템플릿을 둔다.
3. 반복 작업이 불편해지면 NFS 또는 이미지 자동화 도입을 검토한다.
4. VM 수가 늘어나면 Terraform/Ansible/Packer를 단계적으로 추가한다.

[목차로 돌아가기](../README.md)
