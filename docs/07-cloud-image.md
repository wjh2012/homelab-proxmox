# 07. Ubuntu Cloud Image 준비

여러 Ubuntu VM을 반복해서 만들 계획이라면 ISO 설치를 매번 반복하기보다 Ubuntu Cloud Image + Cloud-Init 템플릿을 사용하는 것이 편하다.

## Cloud Image 선택

Ubuntu 24.04 LTS Noble, Intel N100/x86-64 환경에서는 다음 이미지를 사용한다.

```text
noble-server-cloudimg-amd64.img
```

- `amd64`: x86-64
- `arm64`: ARM 서버용이므로 N100에는 사용하지 않는다
- VMDK/OVA/Azure용 이미지는 Proxmox 기본 템플릿 용도로 선택하지 않는다

공식 이미지:
https://cloud-images.ubuntu.com/noble/current/

## Proxmox에 업로드

Web UI에서 `.img` 업로드가 허용되는 경우 `local` 스토리지에 업로드할 수 있다.

업로드된 파일의 실제 경로는 구성에 따라 다르므로 Shell에서 확인한다.

기본 `local`의 ISO 디렉터리를 사용했다면 예:

```bash
ls -lh /var/lib/vz/template/iso/
```

## 템플릿용 VM ID

템플릿 VM ID로 `9000`을 사용하는 것은 필수가 아니다.

예:

```text
100~     일반 VM
9000~    Template
```

처럼 관리자가 보기 쉽게 범위를 나누기 위한 관례일 뿐이다.

클러스터 전체에서 VM ID는 고유해야 한다.

다음 장: [Cloud-Init 템플릿 생성](08-cloud-init-template.md)
