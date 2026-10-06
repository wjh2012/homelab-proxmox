# 06. 스냅샷과 기본 운영

## 스냅샷의 목적

스냅샷은 특정 시점으로 빠르게 롤백하기 위한 기능이다.

백업과는 다르다.

```text
Snapshot = 빠른 롤백
Backup   = 디스크 고장/노드 장애까지 대비한 별도 복구본
```

## 첫 스냅샷

Ubuntu 설치 직후 깨끗한 상태를 남기고 싶다면:

```text
Ubuntu 설치 완료
→ VM 정상 종료
→ Snapshot: fresh-install
```

기준점 스냅샷은 VM을 끈 상태에서 찍으면 가장 단순하고 일관된 상태를 남길 수 있다.

## 기본 업데이트

VM을 다시 시작한 후:

```bash
sudo apt update
sudo apt upgrade -y
```

QEMU Agent 설치:

```bash
sudo apt install -y qemu-guest-agent
sudo systemctl enable --now qemu-guest-agent
```

SSH/네트워크까지 확인한 뒤 두 번째 스냅샷을 남길 수 있다.

```text
Snapshot: base-setup
```

## 언제 스냅샷을 찍는가

권장 예:

- 대규모 패키지 업데이트 전
- 커널 업데이트 전
- Docker/DB 등 주요 소프트웨어 변경 전
- 네트워크/스토리지 설정 변경 전
- OS 버전 업그레이드 전

작은 보안 업데이트마다 항상 스냅샷을 찍을 필요는 없다.

## Proxmox 호스트 재부팅

호스트를 정상 Reboot하면 실행 중 VM을 정리하는 절차가 수행된다.

그래도 유지보수 시 명시적으로 VM을 먼저 정상 종료하고 호스트를 재부팅하면 상태를 확인하기 쉽다.

Ubuntu:

```bash
sudo poweroff
```

Proxmox의 `Stop`은 강제 전원 차단에 가까우므로 정상 종료가 가능한 상황에서는 `Shutdown`을 우선한다.

다음 장: [Ubuntu Cloud Image 준비](07-cloud-image.md)
