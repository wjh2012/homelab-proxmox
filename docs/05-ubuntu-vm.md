# 05. Ubuntu Server VM 생성

## ISO 업로드

Ubuntu Server ISO를 PC에 내려받은 뒤:

```text
Node
→ local
→ ISO Images
→ Upload
```

Checksum을 검증할 경우 Ubuntu가 제공하는 SHA256 값을 사용하는 것을 권장한다.

## General

예:

```text
VM ID: 100
Name:  ubu-dev01
Resource Pool: 비움
Add to HA: OFF
```

VM Name은 Proxmox에서 관리하기 위한 이름이다.

Ubuntu 내부 hostname과 기술적으로 별개지만 관리 편의를 위해 동일하게 맞추는 것을 권장한다.

## OS

업로드한 Ubuntu Server ISO를 선택한다.

## System

일반적인 Ubuntu Server VM 예시:

```text
Machine: q35
BIOS: OVMF (UEFI)
QEMU Agent: ON
SCSI Controller: VirtIO SCSI single
```

`i440fx`도 사용할 수 있다. 단순 서버 VM에서는 충분히 동작한다.

- i440fx: 오래되고 호환성이 높은 가상 머신 구조
- q35: PCIe 기반의 더 현대적인 가상 머신 구조

## CPU

N100 4코어 호스트의 가벼운 개발 VM 예시:

```text
Sockets: 1
Cores:   2
CPU Type: host
```

`host`는 실제 CPU 기능을 게스트에 많이 노출해 성능 면에서 유리하다.

다만 서로 다른 CPU 세대/제조사의 노드 사이 라이브 마이그레이션 호환성이 중요하다면 공통 가상 CPU 모델을 검토한다.

## Memory

16GB 호스트에서 시작값 예시:

```text
Memory: 4096 MiB
```

가벼운 서버는 2GB로도 시작할 수 있다. DB나 빌드 작업 등 용도에 따라 늘린다.

## Disk

일반적인 설정:

```text
Bus/Device: SCSI
Storage: local-lvm
Disk size: 32~64 GiB
Discard: ON
IO Thread: ON
SSD emulation: 실제 저장장치가 SSD/NVMe라면 ON 고려
Cache: Default
```

## Network

```text
Bridge: vmbr0
Model:  VirtIO
```

## Ubuntu 설치 시 Storage

단일 서버 VM이라면:

```text
Use an entire disk: ON
Set up this disk as an LVM group: ON 권장
LUKS Encryption: 필요할 때만
```

Proxmox의 `local-lvm`과 Ubuntu 내부의 LVM은 서로 다른 계층이다.

```text
Proxmox local-lvm
  └─ VM 가상 디스크
       └─ Ubuntu LVM
            └─ ext4 /
```

Ubuntu 설치기가 LVM VG의 일부만 `/`에 할당해 남겨둘 수 있다.

별도의 LV 분리 계획이 없다면 설치 화면에서 `ubuntu-lv`를 늘려 VG의 대부분을 `/`에 할당해도 된다.

## OpenSSH

설치 중 `Install OpenSSH server`를 활성화하면 이후 Proxmox Console 대신 SSH를 사용할 수 있다.

## QEMU Guest Agent 설치

Proxmox에서 QEMU Agent 옵션을 켠 것과 Ubuntu 내부 패키지 설치는 별개다.

Ubuntu 설치 후:

```bash
sudo apt update
sudo apt install -y qemu-guest-agent
sudo systemctl enable --now qemu-guest-agent
```

확인:

```bash
systemctl status qemu-guest-agent
```

다음 장: [스냅샷과 기본 운영](06-snapshot-maintenance.md)
