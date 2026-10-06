# 08. Cloud-Init 템플릿 생성

이 장에서는 VM ID `9000`, 이름 `ubuntu-2404-template`을 예로 든다.

## 1. 빈 VM 생성

Web UI에서:

```text
Create VM

VM ID: 9000
Name: ubuntu-2404-template
OS: Do not use any media
```

예시 설정:

```text
Machine: q35
BIOS: OVMF
SCSI Controller: VirtIO SCSI single
QEMU Agent: ON

CPU: 2 cores
Memory: 2048 MiB

Network:
  Model: VirtIO
  Bridge: vmbr0
```

기본 OS Disk는 만들지 않거나, 생성했다면 이후 제거한다.

## 2. Cloud Image Import

업로드 경로가 다음이라고 가정한다.

```text
/var/lib/vz/template/iso/noble-server-cloudimg-amd64.img
```

Shell:

```bash
qm disk import 9000   /var/lib/vz/template/iso/noble-server-cloudimg-amd64.img   local-lvm
```

완료 후 Hardware에 `Unused Disk 0`가 나타난다.

## 3. OS Disk 연결

`Unused Disk 0`을 연결한다.

```text
Bus/Device: SCSI
Device: scsi0
Discard: ON
IO Thread: ON
```

SSD/NVMe 기반 스토리지라면 SSD emulation을 사용할 수 있다.

## 4. 디스크 크기 확장

Cloud Image 원본 디스크는 작다.

예를 들어 3584 MiB 이미지를 약 32 GiB로 늘리려면 Web UI:

```text
Hardware
→ scsi0
→ Disk Action
→ Resize
```

Resize는 보통 최종 크기가 아니라 **추가할 크기**를 입력한다.

정확히 32 GiB를 목표로 한다면:

```text
32768 MiB - 3584 MiB = 29184 MiB
```

실사용에서는 +29G처럼 단순하게 늘려도 된다.

## 5. CloudInit Drive 추가

```text
Hardware
→ Add
→ CloudInit Drive
→ Storage: local-lvm
```

일반적으로 `ide2`로 추가된다.

## 6. Boot Order

```text
scsi0
ide2
net0
```

핵심은 `scsi0`가 첫 번째 부팅 디스크라는 점이다.

`net0` PXE 부팅이 필요 없다면 비활성화할 수 있다.

## 7. Cloud-Init 설정

Cloud-Init 탭에서 기본값을 설정한다.

예:

```text
User: 원하는 기본 사용자
SSH Public Key: 권장
IP Config: DHCP
DNS: 환경에 맞게
```

Cloud-Init은 Clone VM의 첫 부팅 시 사용자, 네트워크, hostname 등의 초기 설정을 적용한다.

## 8. QEMU Guest Agent 설치

공식 Cloud Image에 `qemu-guest-agent`가 설치되어 있지 않을 수 있다.

템플릿 원본을 한 번 부팅하여 커스터마이징할 경우:

```bash
sudo apt update
sudo apt install -y qemu-guest-agent
sudo systemctl enable --now qemu-guest-agent
```

## 9. 템플릿 정리

원본 VM을 한 번 부팅했다면 템플릿 전환 전에 인스턴스 고유 상태를 정리한다.

```bash
sudo cloud-init clean --logs
sudo truncate -s 0 /etc/machine-id
sudo rm -f /var/lib/dbus/machine-id
sudo rm -f /etc/ssh/ssh_host_*
sudo poweroff
```

`cloud-init clean`을 수행하면 다음 Clone의 첫 부팅에서 Cloud-Init이 다시 초기화 작업을 수행할 수 있다.

Ubuntu Cloud Image의 `/etc/cloud/cloud.cfg`에서 hostname 보존 정책을 확인하려면:

```bash
grep preserve_hostname /etc/cloud/cloud.cfg
```

Cloud-Init이 hostname을 관리하게 하려면 일반적으로 `preserve_hostname: false` 상태가 적합하다.

> 원본 VM을 한 번도 부팅하지 않았다면 위 런타임 정리 작업 자체가 필요하지 않을 수 있다.

## 10. Template 전환

VM이 정지된 상태에서:

```text
VM 우클릭
→ Convert to Template
```

CLI:

```bash
qm template 9000
```

이제 원본을 직접 실행하는 대신 Clone해서 사용한다.

다음 장: [템플릿 복제와 멀티 노드 주의사항](09-template-clone-multinode.md)
