# RTX 4080 SUPER GPU 패스스루 구축 기록

> 작업일: 2026-10-08 · 구성: 단일 Proxmox 호스트 / Ubuntu 24.04 LTS  
> 검증 완료: OVMF 기반 VM 부팅, SSH, Guest Agent, RTX 4080 SUPER PCI 인식  
> **아직 검증하지 않음:** NVIDIA 게스트 드라이버, `nvidia-smi`, CUDA 연산, 장기 안정성

## 환경

| 항목 | 값 |
| --- | --- |
| Proxmox 호스트 | `dev-gpu01.taehwainno.com` (`192.168.0.251`) |
| CPU / RAM | Intel Core i9-14900K / 약 62 GiB |
| GPU | NVIDIA GeForce RTX 4080 SUPER (AD103, `10de:2702`) |
| GPU 오디오 | `10de:22bb` |
| Proxmox 커널 | `7.0.2-6-pve` (점검 당시) |
| VM 네트워크 | `172.16.10.0/24`, `vmbr1` |
| VM IP / Gateway | `172.16.10.203/24`, `172.16.10.1` |
| 성공한 시험 VM | VM ID `103`, 이름 `gpu-test` (최종 `gpu01` 명칭 정리 예정) |
| vCPU / RAM / Disk | 12 (`host`) / 16 GiB / 100 GiB |

이 작업은 기존 N100 홈랩 문서와 별도의 물리 호스트 사례이다.

## 사전 점검 (Proxmox 호스트)

```bash
lspci -nnk -d 10de:
dmesg | grep -Ei 'DMAR|IOMMU'
cat /proc/cmdline
lsmod | grep vfio
```

확인된 장치:

```text
01:00.0 NVIDIA RTX 4080 SUPER [10de:2702]
01:00.1 NVIDIA HD Audio [10de:22bb]
```

IOMMU는 활성화되어 있었으며 GPU와 오디오는 **IOMMU Group 13**에만 함께 있었다. 별도의 ACS override나 `intel_iommu=on` 추가는 진행하지 않았다. 그룹 확인:

```bash
for device in /sys/kernel/iommu_groups/13/devices/*; do
  lspci -nn -s "$(basename "$device")"
done
```

## VM 생성 권장 설정

기존 Ubuntu 24.04 Cloud-Init 템플릿을 Full Clone하거나, UEFI 부팅 가능한 Ubuntu 설치 디스크로 새 VM을 만든다. 이 사례에서는 UEFI를 지원하는 EFI 파티션과 부트로더가 실제 확인되었다. 템플릿을 복제한 경우 **BIOS와 EFI Disk 설정을 명시적으로 확인**한다.

| 구분 | 설정 |
| --- | --- |
| BIOS | **OVMF (UEFI)** |
| Machine | **q35** |
| EFI Disk | 추가, 예: `local-lvm`, EFI type `4m` |
| EFI keys | 시험 VM은 `pre-enrolled-keys=1` 상태로 부팅 성공. Secure Boot 실제 상태는 별도 확인 필요 |
| Display | `Serial terminal 0` (`vga: serial0`) |
| SCSI | `VirtIO SCSI single` |
| CPU | 1 socket × 12 cores, type `host` |
| RAM | 16384 MiB |
| Disk | 100 GiB |
| Network | VirtIO, **bridge `vmbr1`** |
| Cloud-Init | `172.16.10.203/24`, gateway `172.16.10.1` |
| QEMU Guest Agent | 사용 |

기존 VM과 새 VM에 동일 IP를 부여했다면 **동시에 시작하지 않는다**. 실제 VM ID는 `qm list`로 확인한다.

GPU 추가 전 Ubuntu 부팅과 SSH를 먼저 검증한다.

```bash
# Ubuntu 내부
test -d /sys/firmware/efi && echo UEFI || echo Legacy
ip -br addr
ip route
systemctl is-active ssh
cloud-init status --long
```

## GPU PCI 장치 추가

GPU를 추가할 VM만 **정상 종료**한 뒤 Proxmox Web UI:

`VM → Hardware → Add → PCI Device`

| 옵션 | 값 |
| --- | --- |
| Raw Device | GPU `0000:01:00` |
| All Functions | 체크: `01:00.0` GPU + `01:00.1` Audio |
| PCI-Express | 체크 |
| ROM-Bar | 기본값 |
| Primary GPU | 체크 해제 (CUDA 연산용) |

```bash
# Proxmox 호스트: 실제 VM ID를 사용
qm config 103 | grep -E '^(bios|machine|efidisk|hostpci|vga)'
# 확인된 예
# bios: ovmf
# hostpci0: 0000:01:00,pcie=1
# machine: q35
# vga: serial0
```

VM 시작 후 검증:

```bash
# Proxmox 호스트
qm start 103
qm status 103
qm agent 103 ping
lspci -nnk -s 01:00.0
lspci -nnk -s 01:00.1
```

```bash
# Ubuntu VM 내부 (호스트의 qm 명령을 여기서 실행하지 말 것)
lspci -nn -d 10de:
test -d /sys/firmware/efi && echo UEFI || echo Legacy
```

실제 시험 결과: GPU(`10de:2702`)와 오디오(`10de:22bb`)가 Ubuntu 내부에 나타났고 UEFI 부팅, SSH, Guest Agent 응답이 확인되었다. **이는 PCI 인식 성공을 의미하며 CUDA 연산 성공을 뜻하지 않는다.**

## 장애 1: SeaBIOS VM(102)에서 GPU 추가 후 부팅 불가 의심

첫 번째 `gpu01`은 VM ID **102**, `q35` + 기본 **SeaBIOS**로 생성되었다. GPU 연결 전에는 SSH가 됐으나 GPU 연결 뒤에는 다음과 같았다.

- `qm status 102`: `running`
- `qm agent 102 ping`: 응답 없음
- SSH 타임아웃
- 시리얼 콘솔에 로그인 화면이 나타나지 않음

당시 Proxmox 로그:

```text
kvm: warning: vfio_container_dma_map(...) = -22 (Invalid argument)
```

`vfio-pci` GPU/오디오 리셋 로그도 나타났다. 이는 DMA 매핑 경고로 **원인 확정은 불가**하다.

호스트 GPU의 Resizable BAR 1은 16 GiB였고, 관련 sysfs 인터페이스도 확인했다. 그러나 **BAR 크기를 실제 변경하지 않았고 BIOS ReBAR도 변경하지 않았다.** 단순히 오류 매핑 크기가 16 GiB라는 이유로 ReBAR가 원인이라고 단정할 수 없다.

VM 102에서 GPU 설정을 제거하자 Guest Agent와 SSH가 정상 동작했다. 이후 별도의 OVMF 기반 VM 103에서 GPU PCI 인식에 성공했다. **두 VM은 펌웨어 이외 설정도 달라, SeaBIOS 하나만 원인이었다고 증명된 것은 아니다.**

## 장애 2: 새 OVMF VM(103)의 SSH 및 외부 통신 실패

새 `gpu-test`는 Ubuntu GUI 콘솔에서 로그인 가능했으나 SSH 및 외부 통신이 되지 않았다. `ssh.service`는 처음 `inactive`여서 활성화를 점검했고, 서비스가 `active`여도 통신되지 않았다.

```bash
# Ubuntu VM
ip -br addr
ip route
sudo ss -lntp | grep ':22'
ip neigh
```

실제 원인은 **새 VM NIC가 `vmbr0`에 연결되어 있었던 것**이었다. VM의 IP는 `172.16.10.203/24`였으나 올바른 내부 브리지는 `vmbr1`이다.

```text
dev01  (100): bridge=vmbr1
db01   (101): bridge=vmbr1
gpu-test (103): bridge=vmbr0  ← 잘못된 설정
```

Proxmox VM Hardware → Network Device에서 브리지를 `vmbr1`로 수정한 뒤 SSH 연결에 성공했다.

```bash
# 조회 (Proxmox 호스트)
qm config 103 | grep '^net0:'
# VM 정지 상태에서 설정하는 명령 예시. MAC 주소는 실제 값으로 교체
qm set 103 --net0 virtio=<실제-MAC>,bridge=vmbr1,firewall=1
```

## 최종 확인 및 남은 작업

- [x] OVMF + q35 부팅
- [x] 내부 네트워크 `vmbr1`, SSH
- [x] Proxmox Guest Agent 응답
- [x] VFIO GPU/오디오 바인딩
- [x] Ubuntu에서 RTX 4080 SUPER 및 오디오 PCI 장치 인식
- [ ] VM 이름/Ubuntu hostname을 `gpu01`로 정리 (검증 당시 `gpu-test`)
- [ ] NVIDIA 드라이버 설치 후 `nvidia-smi` 확인
- [ ] CUDA 연산 테스트
- [ ] GPU 패스스루 상태에서 재부팅 안정성 확인
- [ ] Docker 및 NVIDIA Container Toolkit (별도 작업)

## 참고

- [Proxmox PCI(e) Passthrough](https://pve.proxmox.com/wiki/PCI_Passthrough)
- [Proxmox VM 관리 및 PCI 패스스루](https://pve.proxmox.com/pve-docs/chapter-qm.html#qm_pci_passthrough)
- [Linux PCI sysfs ABI](https://www.kernel.org/doc/Documentation/ABI/testing/sysfs-bus-pci)
