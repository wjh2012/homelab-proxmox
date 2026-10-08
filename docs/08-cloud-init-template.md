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

원본 VM을 부팅하여 QEMU Guest Agent, SSH, 패키지 등을 설정했다면 **전원을 끄기 직전에** 인스턴스 고유 상태를 정리한다. 이 명령은 **Ubuntu 게스트 VM 내부**에서 실행한다(Proxmox 호스트에서 실행하지 않음).

### 정리 전 확인

```bash
cloud-init --version
cloud-init clean --help
grep -E '^(preserve_hostname|ssh_deletekeys|ssh_genkeytypes):' /etc/cloud/cloud.cfg
```

- `preserve_hostname: false`: Cloud-Init에서 새 hostname을 설정할 수 있도록 하는 일반적인 설정이다. 다른 설정 파일에서 덮어쓸 수도 있으므로 실제 적용 상태를 확인한다.
- `ssh_deletekeys: true`(기본값): 첫 부팅 시 기존 SSH 서버 호스트 키를 제거하고 새 키를 만들도록 한다. 별도로 비활성화하지 않는다.
- `ssh_genkeytypes`: SSH 호스트 키 생성 유형 설정이다. 해당 항목이 출력되지 않는다고 곧바로 비정상은 아니다.
- `cloud-init clean --help`에서 `--machine-id`를 지원하는지 확인한다.

### 초기화 및 종료

```bash
# Cloud-Init 실행 이력과 로그, machine-id 초기화
sudo cloud-init clean --logs --machine-id

# 기존 SSH 서버 호스트 키 삭제 (사용자 authorized_keys와는 별개)
sudo rm -f /etc/ssh/ssh_host_*

# VM 종료
sudo poweroff
```

| 작업 | 이유 |
| --- | --- |
| `cloud-init clean --logs` | 이전 Cloud-Init 인스턴스 캐시/실행 기록과 로그를 정리해 다음 부팅에 다시 초기화를 수행하도록 준비 |
| `--machine-id` | Clone마다 고유한 `/etc/machine-id`가 생성되도록 초기화 |
| `/etc/ssh/ssh_host_*` 삭제 | 여러 VM이 같은 **SSH 서버 호스트 키**를 공유하지 않도록 처리 |
| `poweroff` | 정리된 상태로 종료하여 Template으로 변환 |

`--machine-id`를 지원하는 버전이라면 기존의 `sudo truncate -s 0 /etc/machine-id`를 별도로 실행할 필요가 없다. `/var/lib/dbus/machine-id`가 별도 일반 파일인 특수 구성에서는 별도 확인이 필요하지만, 일반적인 최신 Ubuntu 이미지에서는 보통 `/etc/machine-id`와 연결되어 있다.

**주의사항**

- `/etc/ssh/ssh_host_*`는 SSH **서버 식별용 키**다. 로그인용 `~/.ssh/authorized_keys`를 지우는 명령이 아니다.
- 템플릿에 사용자 개인 SSH 키, 불필요한 `authorized_keys`, 비밀번호 또는 토큰을 남겨두지 않는다. SSH 접근 키는 가급적 Proxmox Cloud-Init 설정으로 Clone마다 주입한다.
- `--configs ssh_config`는 Cloud-Init이 생성한 SSH 설정 정리 옵션이며 SSH 호스트 키 삭제와는 다른 작업이다. SSH 서버 설정을 커스터마이징했다면 무분별하게 사용하지 않는다.
- 정리 후에는 **원본 VM을 다시 부팅하지 말고** 곧바로 Template으로 변환한다. 다시 부팅했다면 종료 전 초기화 절차를 재실행한다.
- 원본 VM을 한 번도 부팅하지 않았다면 런타임 초기화가 필요하지 않을 수 있다.

참고: [Cloud-Init CLI 문서](https://docs.cloud-init.io/en/latest/reference/cli.html), [Cloud-Init SSH 모듈](https://docs.cloud-init.io/en/latest/reference/modules.html#ssh).

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
