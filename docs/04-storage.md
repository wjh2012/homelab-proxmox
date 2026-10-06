# 04. local / local-lvm 스토리지 이해

Proxmox 기본 설치 후 흔히 다음 두 스토리지가 보인다.

```text
local
local-lvm
```

둘은 용도가 다르다.

## local

일반적으로 디렉터리 기반 스토리지다.

주요 용도:

- ISO Images
- Container Templates
- Backup
- Snippets 등

예:

```text
local
└─ ISO Images
   └─ ubuntu-24.04-live-server-amd64.iso
```

기본 경로는 일반적으로 `/var/lib/vz`다.

## local-lvm

기본 설치에서 LVM-thin으로 구성되는 경우가 일반적이다.

주요 용도:

- VM Disk
- LXC Root Disk

예:

```text
local-lvm
├─ vm-100-disk-0
└─ vm-101-disk-0
```

## 왜 용량이 나뉘어 보이는가

Proxmox 설치 시 시스템 디스크 전체를 단일 파일시스템으로 사용하는 것이 아니라, root/local 영역과 VM 디스크용 LVM-thin 영역으로 나눠 구성할 수 있다.

그래서 Web UI에서 `local`과 `local-lvm`의 총 용량이 서로 다르게 보이는 것이 정상이다.

## local-lvm은 공유 스토리지가 아니다

매우 중요하다.

```text
n100-01/local-lvm
≠
n100-02/local-lvm
```

이름이 같아도 각 물리 노드의 로컬 디스크다.

따라서 n100-01의 local-lvm에 있는 VM 디스크나 템플릿 디스크를 n100-02가 직접 읽는 것은 아니다.

이 특성은 이후 템플릿과 VM 마이그레이션 설계에 영향을 준다.

## Discard

VM 디스크에서 `Discard`를 활성화하면 게스트의 TRIM/UNMAP 정보가 하위 스토리지로 전달될 수 있다.

Ubuntu에서 사용하지 않는 블록을 정리할 때 LVM-thin 공간 회수에 도움이 된다.

Ubuntu에서는 다음으로 주기적 TRIM 상태를 확인할 수 있다.

```bash
systemctl status fstrim.timer
```

수동 실행:

```bash
sudo fstrim -av
```

다음 장: [Ubuntu Server VM 생성](05-ubuntu-vm.md)
