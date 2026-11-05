# Firefly AIBOX-K3에 Armbian SPI NOR 부팅 지원 추가하기

**Armbian build framework 기여와 실제 하드웨어 검증 완전 가이드**

- 작성일: 2026-10-19
- 대상 보드: Firefly AIBOX-K3 (SpacemiT K3, 8GB RAM, 128GB UFS)
- Armbian: `26.11.0-trunk`, `legacy` branch, kernel `6.18.3`
- 빌드 호스트 예: `192.168.1.10` (`control-plane-eqr7`)
- 테스트 보드 예: `192.168.1.123` (`firefly-aibox-k3`, 초기 테스트는 `192.168.1.70`)

> **중요:** 이 문서는 배경/설계, 최종 빌드/설치, 검증, 변경/PR, 대안 설치를 모두 담고 있습니다. 빌드/설치만 필요하면 **빠른 시작 → 6장 → 7장 → 19장**을 보세요.
>
> **이 문서의 최종 목표는 커밋 `4646b10de`("Add support for Firefly AIBOX-K3")를 Armbian upstream에 기여하는 것입니다.**
> 커밋이 무엇을 바꾸는지, 왜 그렇게 바꿨는지, upstream 제출 전에 무엇을 정리해야 하는지까지
> 정리한 문서는 **`AIBOX-K3-GUIDE.md`**입니다. 12장(팬/전원관리 문제)은 그 근거 자료이므로
> 길게 남겨두었지만, contrib 관점에서는 3장(소스 선택 근거)만 보면 됩니다.
>
> IP 주소와 `/var/home/...` 경로는 작성 환경의 예시입니다. 다른 환경에서는 자신의 경로와 장치로 바꾸세요.

---

## 목차

0. [입문자용: 무슨 일이 일어난 건가?](#0-입문자용-무슨-일이-일어난-건가)
1. [이 문서의 목적](#1-이-문서의-목적)
2. [한눈에 보는 결론](#2-한눈에-보는-결론)
3. [하드웨어와 부팅 구조](#3-하드웨어와-부팅-구조)
4. [원래 보드 설정 파일](#4-원래-보드-설정-파일)
5. [변경 설계와 근거](#5-변경-설계와-근거)
- [빠른 시작](#빠른-시작)
6. [빌드](#6-빌드)
7. [설치 방식 선택](#7-설치-방식-선택)
8. [검증 결과 요약](#8-검증-결과-요약)
9. [문제 해결](#9-문제-해결)
10. [커밋 메시지와 PR 본문](#10-커밋-메시지와-pr-본문)
11. [부록](#11-부록)
12. [펌웨어 소스 선택 근거 (팬 문제 이력)](#12-펌웨어-소스-선택-근거-팬-문제-이력)
13. [의사 결정 로그](#13-의사-결정-로그)
14. [Armbian만으로 빌드하고 설치하는 최종 절차](#14-armbian만으로-빌드하고-설치하는-최종-절차)
15. [변경 파일과 diff](#15-변경-파일과-diff)
16. [최종 커밋과 PR](#16-최종-커밋과-pr)
17. [클린 설치용 TitanTools 펌웨어 패키지](#17-클린-설치용-titantools-펌웨어-패키지)
18. [최종 부팅 경로와 선택 기록](#18-최종-부팅-경로와-선택-기록)
19. [Armbian installer로 설치](#19-armbian-installer로-설치)

## 빠른 시작

빌드:

```bash
./compile.sh build BOARD=firefly-aibox-k3 BRANCH=legacy \
  BUILD_DESKTOP=no BUILD_MINIMAL=no EXPERT=yes \
  KERNEL_CONFIGURE=no RELEASE=resolute
```

산출물:

```text
output/images/Armbian-unofficial_26.11.0-trunk_Firefly-aibox-k3_resolute_legacy_6.18.3.img
output/debs/linux-u-boot-firefly-aibox-k3-legacy_..._2022.10-Se471-...deb
```

설치 방법:

| 방법 | 대상 | 문서 |
|---|---|---|
| Armbian installer | live USB → UFS + SPI NOR | 19장 |
| TitanTools | UFS + SPI NOR | 17장 |
| 수동 | dd + flashcp | 14.11 |
| fastboot | SPI NOR | 17.6 |

현재 보드처럼 SPI NOR에 Armbian U-Boot가 이미 있으면 **Armbian installer**를 쓸 수 있습니다. 공장 SPI NOR만 있으면 TitanTools/fastboot로 먼저 SPI NOR을 써야 합니다.

---

## 0. 입문자용: 무슨 일이 일어난 건가?

### 0.1 한 줄 요약

AIBOX-K3는 **SPI NOR이라는 작은 펌웨어 칩**에서 부트로더를 읽고, **UFS라는 큰 저장 장치**에서 Linux를 읽습니다.

처음에는 SPI NOR의 공장 펌웨어를 그대로 두고 UFS에만 Armbian을 설치했습니다. 그다음 SPI NOR까지 Armbian 펌웨어로 바꾸려다 **팬 제어가 깨졌고**, 원인을 추적해서 **공장 펌웨어와 같은 Firefly 소스 태그**로 Armbian을 빌드해 해결했습니다.

### 0.2 PC에 비유하면

| AIBOX-K3 | PC에 비유 | 역할 |
|---|---|---|
| SPI NOR | 메인보드의 BIOS/UEFI 칩 | 부트로더와 펌웨어 저장 |
| UFS | SSD | Linux 커널과 루트 파일시스템 |
| BootROM | CPU에 내장된 첫 코드 | SPI NOR에서 FSBL을 찾음 |
| FSBL (SPL) | BIOS의 메모리 초기화 단계 | DRAM 초기화, 다음 단계 로드 |
| OpenSBI | UEFI 런타임/보안 모니터에 가까운 층 | RISC-V M-mode 런타임 |
| U-Boot | GRUB 같은 부트로더 | 커널/DTB/initrd 로드 |
| ESOS | 전력 관리용 별도 마이크로컨트롤러 펌웨어 | 전력/실시간 코어 관리 |
| Device Tree (DTB) | 하드웨어 지도 | “어떤 I2C에 어떤 칩이 있는지” 설명 |
| Kernel | OS 커널 | Linux |
| rootfs | C: 드라이브 | Linux 파일시스템 |
| bootfs | 작은 부팅 파티션 | 커널, DTB, initrd, boot.scr |
| fastboot/TitanTools | USB 복구/플래싱 도구 | UFS/SPI NOR에 이미지 쓰기 |
| MTD | Linux의 플래시 추상화 | SPI NOR을 `/dev/mtdN`으로 보여줌 |
| mtdparts | 플래시 파티션 표 | `bootinfo`, `fsbl`, `env`, ... 구분 |
| I2C | 느린 주변장치 버스 | 팬 컨트롤러, PMIC, RTC 연결 |
| hwmon | Linux 하드웨어 모니터링 | 온도/팬 속도 sysfs |
| thermal zone | Linux 온도 센서 | `thermal_zone0` 등 |

### 0.3 부팅 릴레이

AIBOX-K3는 다음 순서로 부팅합니다.

```text
전원 ON
  |
  v
BootROM (SoC 내부, 수정 불가)
  |
  | SPI NOR의 bootinfo를 읽음
  v
bootinfo (FSBL의 위치/크기)
  |
  v
FSBL (U-Boot SPL, DRAM 초기화)
  |
  v
OpenSBI (M-mode 런타임)
  |
  v
U-Boot (부트로더)
  |
  | boot.scr 또는 env_k3.txt를 읽음
  v
Linux kernel + DTB + initrd
  |
  v
rootfs
```

이것은 **릴레이 경주**와 같습니다. 각 주자는 다음 주자에게 정해진 형식의 바통을 넘겨야 합니다. 한 주자만 다른 버전이어도 바통이 떨어집니다.

이번에 문제가 된 “바통”은 **MPXY**라는 전력 관리 메일박스입니다.

### 0.4 저장 장치가 두 개인 이유

AIBOX-K3에는 저장 장치가 두 개 있습니다.

```text
SPI NOR (8MiB)                UFS (128GB)
┌─────────────────┐          ┌─────────────────┐
│ bootinfo        │          │ bootfs (256MiB) │
│ fsbl            │          │  Image          │
│ env             │          │  uInitrd        │
│ esos            │          │  dtb/           │
│ opensbi         │          │  env_k3.txt     │
│ uboot           │          │  boot.scr       │
└─────────────────┘          │ rootfs (나머지)  │
                             └─────────────────┘
```

- **SPI NOR**: 작고, BootROM이 가장 먼저 읽습니다. 부트로더/펌웨어 저장.
- **UFS**: 크고, Linux가 사용합니다. 커널/루트 파일시스템 저장.

Armbian 이미지는 원래 UFS에만 쓰입니다. SPI NOR은 공장 펌웨어가 그대로 남습니다.

### 0.5 처음에는 왜 잘 됐나

처음에는 다음 상태였습니다.

```text
SPI NOR: 공장 U-Boot + 공장 OpenSBI + 공장 ESOS
UFS:     Armbian bootfs + Armbian rootfs
```

공장 U-Boot는 `env_k3.txt`를 읽고 UFS의 `Image`와 DTB를 로드합니다.

```text
console=ttyS0,115200
knl_name=Image
dtb_dir=dtb/spacemit
```

그래서 Armbian 커널이 정상 부팅됐고, 팬도 공장 펌웨어가 제어했습니다.

### 0.6 SPI NOR을 Armbian 것으로 바꾸면 왜 팬이 최대가 됐나

SPI NOR의 U-Boot/OpenSBI/ESOS를 Armbian 것으로 바꾸면, Linux는 계속 부팅되지만 **팬 제어가 깨질 수 있습니다**.

팬은 `CTF2301`이라는 별도 칩이 제어합니다.

```text
Linux
  -> i2c6
    -> ctf2301@4c
      -> 팬 PWM
```

그런데 i2c6을 쓰려면 전력 관리 파워 도메인이 필요하고, 그 파워 도메인은 MPXY 메일박스를 통해 펌웨어와 통신합니다.

```text
팬 -> i2c6 -> power-controller -> mpxy_mbox -> SBI MPXY -> OpenSBI/ESOS
```

Armbian이 사용한 `k3-br-v1.0.8` 펌웨어 세트에서는 이 체인이 끊어졌습니다.

```text
riscv-sbi-mpxy-mbox soc:mpxy_mbox@0: SBI MPXY extension not available
```

그 결과:

```text
mpxy_mbox 실패
  -> power-controller defer
    -> i2c6 defer
      -> ctf2301 probe 안 됨
        -> 팬은 드라이버가 없을 때의 기본값 = 최대 속도
```

온도는 30~35°C로 정상이었습니다. **과열이 아니라 팬 제어 드라이버가 붙지 않은 것**입니다.

### 0.7 왜 335°C가 나왔나

`CTF2301`은 팬 컨트롤러이면서 자체 온도 센서도 제공합니다. Linux에서는 `hwmon0`으로 등록됩니다.

```text
/sys/class/hwmon/hwmon0/name = ctf2301
/sys/class/hwmon/hwmon0/temp1_input = 345625
```

이 값을 그대로 읽으면 345°C로 보입니다. 하지만 이 값은 **CPU 온도가 아니고**, 스케일도 SoC 온도 센서와 다릅니다.

실제 SoC 온도는 `thermal_zone0`입니다.

```text
/sys/class/thermal/thermal_zone0/temp = 35000  -> 35°C
```

Armbian welcome 메시지가 `hwmon0`을 CPU 온도로 읽어서 335°C로 표시했습니다. `thermal_zone0`을 읽도록 고치면 35°C로 정상 표시됩니다.

### 0.8 그래서 무엇을 했나 (의사 결정 요약)

팬 문제를 보고 세 가지 선택지가 있었습니다.

| 선택지 | 내용 | 결과 |
|---|---|---|
| A | 공장 SPI NOR로 되돌리고 Armbian bootfs/rootfs만 사용 | 팬은 정상, 하지만 Armbian-only가 아님 |
| B | Armbian 펌웨어 세트를 고쳐서 팬까지 정상화 | **선택** |
| C | 공장 FSBL/OpenSBI/ESOS + Armbian U-Boot 하이브리드 | 버전 혼합이라 위험 |

B를 선택했습니다. 그리고 중요한 사실을 찾았습니다.

- 공장 펌웨어는 `spacemit-com`의 `k3-br-v1.0.8`이 아니라
- **Firefly의 `k3/linux6.18_release_v1.0.0a`** 태그를 사용합니다.
- 공장 U-Boot 커밋은 `e471ad131b`, OpenSBI는 `842d995`, ESOS는 SDK 빌드입니다.

그래서 AIBOX-K3 보드에서만 Firefly 태그를 쓰도록 하고, ESOS blob도 SDK 버전으로 맞췄습니다. 그 뒤 MPXY가 정상 동작했고, `CTF2301` 팬 컨트롤러가 붙었고, 팬이 조용해졌습니다.

### 0.9 최종 상태

```text
SPI NOR: Armbian이 Firefly 태그로 빌드한 U-Boot/OpenSBI + Firefly ESOS blob
UFS:     Armbian bootfs/rootfs
부팅:    SPI NOR U-Boot -> autoboot/bootfunc -> boot.scr -> armbianEnv.txt -> Image/DTB -> Linux
팬:      CTF2301 fan control 정상 (PWM 16~60/255, 약 550~1,440 RPM)
온도:    thermal_zone0 35~39°C
```

### 0.10 이 문서를 읽는 방법

- **처음 읽는 분**: `AIBOX-K3-GUIDE.md` → 0장 → 1장 → 2장
- **보드 설정이 궁금한 분**: 4장 → 5장 → 15장
- **빌드/설치만 하고 싶은 분**: 빠른 시작 → 6장 → 7장 → 19장
- **팬/온도 문제가 궁금한 분**: 12장 → 13장 (근거 이력)
- **PR을 올릴 분**: `AIBOX-K3-GUIDE.md` 9장 → 10.3 → 16장

---

## 1. 이 문서의 목적

이 문서는 Firefly AIBOX-K3에 Armbian을 설치하면서, 단순히 **bootfs/rootfs만 UFS에 쓰는 단계**에서 한 단계 더 나아가 **SPI NOR에 Armbian U-Boot를 설치하고, 그 U-Boot로 Armbian 이미지를 부팅**하는 과정을 정리한 문서입니다.

처음 접하는 사람도 따라할 수 있도록 다음을 모두 설명합니다.

- 원래 Armbian 보드 설정 파일이 무엇이고, 각 줄이 무슨 역할을 하는지
- 왜 그대로는 SPI NOR 부팅이 안 되는지
- 실제 AIBOX-K3에서 무엇을 확인했고, 그 증거가 무엇을 의미하는지
- 왜 이런 방식으로 설정을 바꿨는지
- Armbian build framework로 이미지를 빌드하는 방법
- TitanTools, U-Boot fastboot, Linux 명령으로 플래싱하는 방법
- 실패했을 때 복구하는 방법
- Armbian 프로젝트에 올릴 커밋 메시지와 PR 본문

> 이 문서는 실제로 빌드하고, 실제 보드에 쓰고, 재부팅해서 확인한 결과를 바탕으로 작성했습니다. 일부 명령은 실행한 그대로이며, IP 주소와 파일 경로는 그때의 환경입니다.

---

## 2. 한눈에 보는 결론

### 2.1 최종 상태

| 항목 | 결과 |
|---|---|
| 보드 | Firefly AIBOX-K3 |
| SoC | SpacemiT K3 (RISC-V) |
| RAM | 8GB LPDDR5 |
| 저장 장치 | 128GB UFS |
| SPI NOR | 8MiB (gd25lq64c) |
| SPI NOR U-Boot | `2022.10_armbian-2022.10-Se471-P78cf-Hde31-V0a62-B56a7-R448a` (Firefly `k3/linux6.18_release_v1.0.0a`) |
| UFS bootfs | 256MiB FAT, `/dev/sda1`, label `armbi_boot` |
| UFS rootfs | ext4, `/dev/sda2`, label `armbi_root` |
| Kernel | `6.18.3-legacy-spacemit-k3` |
| 부팅 경로 | SPI NOR U-Boot → `autoboot`/`bootfunc` → `/boot.scr` → `armbianEnv.txt` → `Image`/DTB → Linux |
| 팬 | `ctf2301_fan`, PWM 16~60/255, 약 550~1,440 RPM (온도에 따라 변동) |
| CPU 온도 | `thermal_zone0` 35~39°C |
| 검증 | SPI NOR 6개 파티션 MATCH, MPXY 정상, 팬 제어 정상, `boot.scr` 부팅, `systemctl --failed` 0 |
| 복구 | Boot 버튼 + USB-C + TitanTools/fastboot |

### 2.2 핵심 변경 (1차 + 2차 + 3차)

1. **`boot.scr`과 `armbianEnv.txt` 생성**
   - Armbian U-Boot는 `boot.scr`/`extlinux`/EFI를 찾습니다.
   - 기존 이미지에는 `boot.scr`이 없고 `env_k3.txt` 경로만 있어서 공장 U-Boot에서만 부팅됐습니다.

2. **Linux에 SPI NOR 파티션 노출**
   - `mtdparts`를 `extraargs`에 추가했습니다.
   - 이게 없으면 `/dev/mtd1..6`이 생기지 않아 SPI NOR을 쓸 수 없습니다.

3. **`write_uboot_platform_mtd()` 추가**
   - SPI NOR의 `bootinfo`, `fsbl`, `env`, `esos`, `opensbi`, `uboot`를 씁니다.
   - `mtdN` 번호가 아니라 **파티션 이름**으로 찾습니다.

4. **(2차) Firefly 펌웨어 태그 + ESOS + 온도 수정**
   - 공장과 같은 `k3/linux6.18_release_v1.0.0a` 태그로 U-Boot/OpenSBI를 빌드합니다.
   - ESOS blob을 SDK 버전(506,499 bytes)으로 맞춥니다.
   - CPU 온도는 `thermal_zone0`을 사용합니다.

5. **(3차) U-Boot 001~003 rebase + distro boot 전환**
   - `001`~`003` 패치를 Firefly 태그 기준으로 다시 작성했습니다.
   - U-Boot env를 vendor env에서 Armbian `autoboot`/`bootfunc` env로 교체했습니다.
   - 최종 부팅 경로는 `/boot.scr` → `armbianEnv.txt`입니다.

### 2.3 실제 부팅 로그의 핵심

> 아래는 최종 구현(`...P78cf-Hde31-V0a62...`, `boot.scr` 경로)의 실제 시리얼 로그입니다.

```text
U-Boot SPL 2022.10_armbian-2022.10-Se471-P78cf-Hde31-V0a62-B56a7-R448a
U-Boot 2022.10_armbian-2022.10-Se471-P78cf-Hde31-V0a62-B56a7-R448a

Loading K3 environment
Loading Armbian environment (scsi 0:1)
Booting Armbian (scsi 0:1)
Starting kernel ...
...
Welcome to Armbian-unofficial 26.11.0-trunk resolute!
...
firefly-aibox-k3 login:
```

즉 **공장 SPI NOR U-Boot를 Armbian U-Boot로 교체**하고, **UFS의 Armbian 이미지를 `boot.scr`로 부팅**하는 데 성공했습니다.

---

## 3. 하드웨어와 부팅 구조

### 3.1 AIBOX-K3의 저장 장치

AIBOX-K3에는 두 종류의 저장 장치가 있습니다.

| 저장 장치 | 용도 | 크기 | Linux 장치 |
|---|---|---|---|
| SPI NOR | 부트로더와 펌웨어 | 8MiB | `/dev/mtd0..6` |
| UFS | 운영체제 (bootfs/rootfs) | 128GB | `/dev/sda` |

중요한 점은 **SPI NOR과 UFS의 역할이 다르다**는 것입니다.

- SPI NOR: BootROM이 처음 읽는 부트로더 저장소
- UFS: Linux 커널과 루트 파일시스템

Armbian 이미지는 기본적으로 UFS에만 쓰입니다. SPI NOR은 공장 펌웨어가 그대로 남아 있습니다. 이 문서의 작업은 **SPI NOR도 Armbian 것으로 교체**하는 것입니다.

### 3.2 전체 부팅 체인

AIBOX-K3의 부팅 순서는 다음과 같습니다.

```text
전원 ON
  |
  v
BootROM (SoC 내부, 수정 불가)
  |
  | SPI NOR offset 0의 bootinfo를 읽음
  v
bootinfo (128KiB)
  |
  | FSBL.bin의 위치와 크기를 알려줌
  v
FSBL.bin (U-Boot SPL, 512KiB 파티션)
  |
  | DRAM 초기화
  | ESOS, OpenSBI, U-Boot을 DRAM에 로드
  v
ESOS (1MiB)  +  OpenSBI (384KiB)
  |
  | OpenSBI가 U-Boot으로 제어를 넘김
  v
U-Boot (약 5.9MiB 파티션)
  |
  | boot.scr을 찾아 실행
  v
boot.scr (UFS bootfs)
  |
  | armbianEnv.txt 로드
  | Image, uInitrd, DTB 로드
  v
Linux kernel
```

각 단계의 역할:

- **BootROM**: SoC에 구워진 첫 단계 코드입니다. SPI NOR의 `bootinfo`를 읽고 FSBL을 찾습니다.
- **bootinfo**: FSBL의 offset/size를 담은 80바이트 구조체입니다. 이 파일이 깨지면 보드가 부팅하지 못합니다.
- **FSBL (First Stage Boot Loader)**: U-Boot SPL입니다. DRAM을 초기화하고 다음 단계를 로드합니다.
- **ESOS**: SpacemiT의 전력 관리/실시간 코어용 펌웨어입니다.
- **OpenSBI**: RISC-V의 M-mode 런타임입니다. Linux의 S-mode와 하드웨어 사이를 중계합니다.
- **U-Boot**: 운영체제를 로드하는 부트로더입니다. Armbian U-Boot는 `boot.scr`을 찾습니다.
- **boot.scr**: U-Boot이 실행하는 스크립트입니다. `armbianEnv.txt`를 읽고 커널/initrd/DTB를 로드합니다.
- **Linux**: 최종 운영체제입니다.

### 3.3 SPI NOR 파티션 레이아웃

실제 보드의 `/proc/mtd`에서 확인한 값입니다.

```text
dev:    size   erasesize  name
mtd0: 00800000 00001000 "d420c000.spi"   <- master (전체 8MiB)
mtd1: 00020000 00001000 "bootinfo"       <- 128KiB
mtd2: 00080000 00001000 "fsbl"           <- 512KiB
mtd3: 00010000 00001000 "env"            <- 64KiB
mtd4: 00100000 00001000 "esos"           <- 1MiB
mtd5: 00060000 00001000 "opensbi"        <- 384KiB
mtd6: 005f0000 00001000 "uboot"          <- 5.9MiB
```

`mtdparts` 문자열:

```text
mtdparts=d420c000.spi:128K@0(bootinfo),512K@128K(fsbl),64K@640K(env),1M@704K(esos),384K@1728K(opensbi),-@2112K(uboot)
```

이 문자열은 다음 파일들에서 확인할 수 있습니다.

- 공장 SPI NOR의 U-Boot env
- U-Boot `k3_defconfig`의 `CONFIG_MTDPARTS_DEFAULT`
- `packages/blobs/riscv64/spacemit-k3/partition_4M.json`

`partition_4M.json`의 내용과 같은 layout입니다.

| 파티션 | offset | size | 내용 |
|---|---:|---:|---|
| bootinfo | 0 | 128K | BootROM용 FSBL 위치 정보 |
| fsbl | 128K | 512K | U-Boot SPL |
| env | 640K | 64K | U-Boot 환경 변수 |
| esos | 704K | 1M | ESOS 펌웨어 |
| opensbi | 1728K | 384K | OpenSBI |
| uboot | 2112K | 나머지 (약 5.9M) | U-Boot proper |

### 3.4 UFS 파티션 레이아웃

Armbian 이미지는 GPT를 사용하고, 4096바이트 섹터를 씁니다.

| 파티션 | offset | size | 파일시스템 | 이름 |
|---|---:|---:|---|---|
| bootfs | 12MiB | 256MiB | FAT | `bootfs` |
| rootfs | 268MiB | 나머지 | ext4 | `rootfs` |

Linux에서는 다음과 같이 보입니다.

```text
sda1  256M  vfat  armbi_boot  bootfs  /boot
sda2  119G  ext4  armbi_root  rootfs  /
```

`/boot`는 FAT 파티션이고, rootfs는 ext4입니다.

### 3.5 공장 부팅과 Armbian 부팅의 차이

#### 공장 U-Boot

공장 U-Boot의 env에는 다음이 있습니다.

```text
bootcmd=run autoboot; echo "run autoboot"
autoboot=if test ${boot_device} = nand; then ... elif test ${boot_device} = nor; then run nor_boot; ...
nor_boot=... run set_nor_args; ... run boot_kernel; ...
set_nor_args=setenv bootargs "${bootargs}" mtdparts=${mtdparts} rootfstype=ext4
```

공장 U-Boot는 `env_k3.txt`를 읽고 `Image`와 DTB를 직접 로드합니다.

```text
console=ttyS0,115200
bootdelay=5
knl_name=Image
dtb_dir=dtb/spacemit
loglevel=8
```

#### Armbian U-Boot

Armbian U-Boot의 env는 distro boot 방식입니다.

```text
bootcmd=run autoboot;
autoboot=... run bootfunc; ...
bootfunc=if test -e ${devtype} ${devnum}:${distro_bootpart} /extlinux/extlinux.conf; then ...
         elif test -e ... /boot.scr; then ...
```

즉 `/boot.scr` 또는 `/extlinux/extlinux.conf` 또는 EFI를 찾습니다. `env_k3.txt`만으로는 부팅하지 않습니다.

#### 그래서 둘 다 넣는다

새 이미지에는 다음이 모두 들어 있습니다.

- `env_k3.txt`: 공장 U-Boot용
- `boot.scr`, `armbianEnv.txt`: Armbian U-Boot용

이렇게 하면 **공장 SPI NOR에서도 부팅**되고, **Armbian SPI NOR에서도 부팅**됩니다. 이것이 안전한 전환의 핵심입니다.

---

## 4. 원래 보드 설정 파일

> **읽기 순서:** 4~11장은 1차 구현입니다. 공장 SPI NOR을 유지하고 Armbian bootfs/rootfs만 쓰는 단계, 그리고 SPI NOR에 Armbian U-Boot를 쓰는 기능을 추가하는 과정입니다. 팬/MPXY/온도 문제의 최종 해결은 12장, 최종 빌드/플래싱 절차는 14장을 보세요.

### 4.1 파일 위치

```text
config/boards/firefly-aibox-k3.wip
```

Armbian build framework가 보드를 빌드할 때 이 파일을 읽습니다. 파일 확장자 `.wip`는 Work In Progress, 즉 아직 정식 지원 전인 보드라는 뜻입니다.

### 4.2 원본 전체

```bash
# SpacemiT K3 16-core RISC-V SoC, UFS Storage

BOARD_NAME="AIBOX-K3"
BOARD_VENDOR="firefly"
BOARDFAMILY="spacemit-k3"

# SpacemiT 6.18 vendor BSP is currently classified as legacy in the family config
KERNEL_TARGET="legacy"
KERNEL_TEST_TARGET="legacy"

BOOT_FDT_FILE="spacemit/k3-firefly-aibox-k3.dtb"

IMAGE_PARTITION_TABLE="gpt"
SECTOR_SIZE="4096"
OFFSET="12"

BOOTFS_TYPE="fat"
BOOTSIZE="256"

function post_config_uboot_target__firefly_aibox_k3() {
	write_uboot_platform() {
		echo "NOTE: U-Boot is stored in SPI NOR and is not written to the UFS image."
	}
}

function pre_umount_final_image__firefly_aibox_k3_boot() {
	local boot_dir="${MOUNT}/boot"

	cat > "${boot_dir}/env_k3.txt" <<-'EOF'
	console=ttyS0,115200
	bootdelay=5
	knl_name=Image
	dtb_dir=dtb/spacemit
	loglevel=8
	EOF
}
```

### 4.3 각 변수의 의미

| 변수 | 의미 |
|---|---|
| `BOARD_NAME` | 보드 이름. 로그인 배너, hostname 등에 사용 |
| `BOARD_VENDOR` | 제조사 |
| `BOARDFAMILY` | 보드가 속한 family. `spacemit-k3` 설정을 사용 |
| `KERNEL_TARGET` | 빌드할 kernel branch. 여기서는 `legacy` |
| `KERNEL_TEST_TARGET` | 테스트용 kernel branch |
| `BOOT_FDT_FILE` | U-Boot/Linux가 사용할 DTB 경로 |
| `IMAGE_PARTITION_TABLE` | 파티션 테이블 종류. GPT |
| `SECTOR_SIZE` | 이미지의 논리 섹터 크기. 4096 |
| `OFFSET` | 첫 파티션 전까지의 여유 공간 (MiB) |
| `BOOTFS_TYPE` | `/boot` 파일시스템. FAT |
| `BOOTSIZE` | `/boot` 파티션 크기 (MiB) |

### 4.4 `post_config_uboot_target__firefly_aibox_k3`

```bash
function post_config_uboot_target__firefly_aibox_k3() {
	write_uboot_platform() {
		echo "NOTE: U-Boot is stored in SPI NOR and is not written to the UFS image."
	}
}
```

이 함수는 U-Boot 패키지의 `write_uboot_platform()`을 덮어씁니다.

- 기본 동작: U-Boot을 UFS/SD의 시작 부분에 씁니다.
- 이 보드: U-Boot은 SPI NOR에 있으므로 UFS에 쓰면 안 됩니다.
- 그래서 “쓰지 않는다”는 메시지만 출력합니다.

문제는 이 stub이 **SPI NOR에 쓰는 함수는 제공하지 않는다**는 점입니다. 그래서 `write_uboot_platform_mtd()`를 추가해야 했습니다.

### 4.5 `pre_umount_final_image__firefly_aibox_k3_boot`

```bash
function pre_umount_final_image__firefly_aibox_k3_boot() {
	local boot_dir="${MOUNT}/boot"

	cat > "${boot_dir}/env_k3.txt" <<-'EOF'
	console=ttyS0,115200
	bootdelay=5
	knl_name=Image
	dtb_dir=dtb/spacemit
	loglevel=8
	EOF
}
```

이 함수는 이미지의 `/boot/env_k3.txt`를 만듭니다. 공장 U-Boot가 이 파일을 읽고 커널을 로드합니다.

- `knl_name=Image`: 로드할 커널 파일 이름
- `dtb_dir=dtb/spacemit`: DTB 디렉터리
- `console=ttyS0,115200`: 시리얼 콘솔
- `bootdelay=5`: 부팅 지연

### 4.6 원본의 한계

원본 설정은 **공장 SPI NOR U-Boot에서 Armbian 이미지를 부팅**하는 데는 충분했습니다. 하지만 **Armbian U-Boot로 교체**하는 데는 부족했습니다.

1. **`boot.scr`이 없음**
   - Armbian U-Boot는 `boot.scr`/`extlinux`/EFI를 찾습니다.
   - 원본 이미지의 부팅 설정은 `env_k3.txt`가 전부입니다.

2. **`armbianEnv.txt`에 `fdtfile`/`extraargs`가 없음**
   - `boot.scr`은 `fdtfile`을 사용해 DTB를 찾습니다.
   - `extraargs`가 없으면 `mtdparts`를 커널에 넘길 수 없습니다.

3. **`mtdparts`가 없음**
   - 공장 U-Boot는 `set_nor_args`로 `mtdparts`를 넘깁니다.
   - Armbian U-Boot의 `boot.scr`은 그렇게 하지 않습니다.
   - 그래서 Linux에 `/dev/mtd1..6`이 생기지 않습니다.

4. **`mtd-utils`가 없음**
   - `flashcp`가 없어서 SPI NOR에 쓸 수 없습니다.

5. **`write_uboot_platform_mtd()`가 없음**
   - `platform_install.sh`에는 stub만 있습니다.
   - `armbian-install`이 SPI NOR에 U-Boot을 쓸 방법이 없습니다.

### 4.7 실제 장비에서 확인한 증거

#### `/proc/cmdline`

공장 SPI NOR로 부팅했을 때:

```text
mtdparts=d420c000.spi:128K@0(bootinfo),512K@128K(fsbl),64K@640K(env),1M@704K(esos),384K@1728K(opensbi),-@2112K(uboot)
rootfstype=ext4
root=PARTUUID=...
boot_mode=nor
```

`mtdparts`가 이미 있습니다. 공장 U-Boot의 `set_nor_args`가 넣은 것입니다.

#### `/proc/mtd`

```text
mtd0: 00800000 00001000 "d420c000.spi"
mtd1: 00020000 00001000 "bootinfo"
mtd2: 00080000 00001000 "fsbl"
mtd3: 00010000 00001000 "env"
mtd4: 00100000 00001000 "esos"
mtd5: 00060000 00001000 "opensbi"
mtd6: 005f0000 00001000 "uboot"
```

`mtd0`이 master이고, 실제 파티션은 `mtd1..6`입니다.

#### `/sys/class/mtd/mtd*/name`

```text
/sys/class/mtd/mtd0 name=d420c000.spi
/sys/class/mtd/mtd1 name=bootinfo
/sys/class/mtd/mtd2 name=fsbl
/sys/class/mtd/mtd3 name=env
/sys/class/mtd/mtd4 name=esos
/sys/class/mtd/mtd5 name=opensbi
/sys/class/mtd/mtd6 name=uboot
```

이 이름들이 `write_uboot_platform_mtd()`의 기준입니다.

#### `/boot` 목록

원본 Armbian 이미지:

```text
Image
uInitrd
dtb/spacemit/k3-firefly-aibox-k3.dtb
env_k3.txt
armbianEnv.txt
```

하지만 `boot.scr`과 `extlinux.conf`는 없었습니다.

#### U-Boot 버전

공장 SPI NOR:

```text
U-Boot 2022.10(u-boot commit id: e471ad131bc9)(sdk version: manifest.xml)
```

UFS에 설치된 Armbian U-Boot 패키지:

```text
U-Boot 2022.10_armbian-2022.10-S3243-P8c42-Hb794-Vda9b-B56a7-R448a
```

두 버전이 다르므로, SPI NOR은 아직 공장 U-Boot이라는 것을 알 수 있습니다.

#### U-Boot env

공장 SPI NOR의 env:

```text
bootcmd=run autoboot; echo "run autoboot"
autoboot=... run nor_boot ...
nor_boot=... run set_nor_args ... run boot_kernel ...
set_nor_args=setenv bootargs "${bootargs}" mtdparts=${mtdparts} rootfstype=ext4
```

Armbian U-Boot 패키지의 env:

```text
bootcmd=run autoboot;
autoboot=... run bootfunc ...
bootfunc=... /boot.scr ...
```

이 차이가 “공장 U-Boot는 env_k3.txt, Armbian U-Boot는 boot.scr”이라는 결론의 근거입니다.

#### 파티션 해시 비교

공장 SPI NOR과 Armbian U-Boot 패키지의 파일을 비교했습니다.

| 파티션 | 공장 SPI NOR | Armbian 패키지 | 결과 |
|---|---|---|---|
| bootinfo | 공장 | `bootinfo_spinor.bin` | MATCH |
| fsbl | 공장 | `FSBL.bin` | DIFF |
| env | 공장 | `u-boot-env-default.bin` | DIFF |
| esos | 공장 | `esos.itb` | DIFF |
| opensbi | 공장 | `fw_dynamic.itb` | DIFF |
| uboot | 공장 | `u-boot.itb` | DIFF |

`bootinfo`는 같고, 나머지는 다릅니다. 즉 SPI NOR은 공장 상태이고, Armbian U-Boot은 아직 쓰지 않은 상태입니다.

---

## 5. 변경 설계와 근거

### 5.1 `BOOTSCRIPT`와 `BOOTENV_FILE`

추가한 줄:

```bash
BOOTENV_FILE="spacemit.txt"
BOOTSCRIPT="boot-spacemit.cmd:boot.cmd"
```

#### `BOOTENV_FILE`

`config/bootenv/spacemit.txt`를 이미지의 `/boot/armbianEnv.txt`로 복사합니다.

`spacemit.txt` 내용:

```text
verbosity=1
console=both
overlay_prefix=k1
```

빌드 과정에서 `BOOT_FDT_FILE`이 `fdtfile=`로 추가됩니다.

```text
fdtfile=spacemit/k3-firefly-aibox-k3.dtb
```

#### `BOOTSCRIPT`

`config/bootscripts/boot-spacemit.cmd`를 `/boot/boot.cmd`로 복사하고, `mkimage`로 `/boot/boot.scr`을 만듭니다.

`boot.scr`이 하는 일:

1. `armbianEnv.txt`를 읽습니다.
2. `fdtfile`로 DTB를 로드합니다.
3. `Image`와 `uInitrd`를 로드합니다.
4. `root=PARTUUID=...`와 `extraargs`로 bootargs를 만듭니다.
5. 커널을 시작합니다.

#### 왜 필요한가

Armbian U-Boot의 `bootfunc`는 다음 순서로 찾습니다.

```text
/extlinux/extlinux.conf
/boot/extlinux/extlinux.conf
/boot.scr
/boot/boot.scr
EFI/BOOT/BOOTRISCV64.EFI
```

원본 이미지에는 이 중 어느 것도 없었습니다. 그래서 `boot.scr`을 추가했습니다.

### 5.2 `mtdparts`

추가한 hook:

```bash
function pre_install_kernel_debs__extra_boot_args() {
	display_alert "$BOARD" "Add extra boot arguments" "info"
	run_host_command_logged "echo 'extraargs=net.ifnames=0 mtdparts=d420c000.spi:128K@0(bootinfo),512K@128K(fsbl),64K@640K(env),1M@704K(esos),384K@1728K(opensbi),-@2112K(uboot)' >> '${SDCARD}/boot/armbianEnv.txt'"
	return 0
}
```

이 hook은 `/boot/armbianEnv.txt`에 다음 줄을 추가합니다.

```text
extraargs=net.ifnames=0 mtdparts=d420c000.spi:128K@0(bootinfo),512K@128K(fsbl),64K@640K(env),1M@704K(esos),384K@1728K(opensbi),-@2112K(uboot)
```

`boot.scr`은 `${extraargs}`를 `bootargs`에 넣습니다.

#### 근거 1: 공장 U-Boot env

공장 env의 `set_nor_args`:

```text
set_nor_args=setenv bootargs "${bootargs}" mtdparts=${mtdparts} rootfstype=ext4
```

공장 U-Boot는 `mtdparts`를 자동으로 넣습니다. Armbian U-Boot의 `boot.scr`은 그렇게 하지 않습니다.

#### 근거 2: U-Boot defconfig

`k3_defconfig`:

```text
CONFIG_MTDIDS_DEFAULT="nor0=d420c000.spi"
CONFIG_MTDPARTS_DEFAULT="d420c000.spi:128K@0(bootinfo),512K@128K(fsbl),64K@640K(env),1M@704K(esos),384K@1728K(opensbi),-@2112K(uboot)"
```

U-Boot 자체는 이 layout을 알고 있습니다. 하지만 Linux에 넘기려면 kernel command line에 `mtdparts=`가 필요합니다.

#### 근거 3: `partition_4M.json`

```json
{
  "format": "mtd",
  "partitions": [
    { "name": "bootinfo", "offset": "0", "size": "128K" },
    { "name": "fsbl", "offset": "128K", "size": "512K" },
    { "name": "env", "offset": "640K", "size": "64K" },
    { "name": "esos", "offset": "704K", "size": "1M" },
    { "name": "opensbi", "offset": "1728K", "size": "384K" },
    { "name": "uboot", "offset": "2112K", "size": "3M" }
  ]
}
```

같은 offset/layout입니다.

#### 근거 4: 실제 dmesg

```text
[    1.607325] 6 cmdlinepart partitions found on MTD device d420c000.spi
[    1.612464] Creating 6 MTD partitions on "d420c000.spi":
[    1.617751] 0x000000000000-0x000000020000 : "bootinfo"
[    1.623702] 0x000000020000-0x0000000a0000 : "fsbl"
[    1.628477] 0x0000000a0000-0x0000000b0000 : "env"
[    1.633150] 0x0000000b0000-0x0000001b0000 : "esos"
[    1.637906] 0x0000001b0000-0x000000210000 : "opensbi"
[    1.642948] 0x000000210000-0x000000800000 : "uboot"
```

#### `CONFIG_MTD_PARTITIONED_MASTER`

K3 kernel config:

```text
CONFIG_MTD_CMDLINE_PARTS=y
CONFIG_MTD_BLOCK=y
CONFIG_MTD_PARTITIONED_MASTER=y
CONFIG_MTD_SPI_NOR=y
```

`CONFIG_MTD_PARTITIONED_MASTER=y` 때문에 master가 `mtd0`으로 먼저 등록됩니다. 그래서 파티션 번호가 `mtd1..6`입니다.

### 5.3 `PACKAGE_LIST_BOARD="mtd-utils"`

추가한 줄:

```bash
PACKAGE_LIST_BOARD="mtd-utils"
```

`flashcp`, `flash_erase`, `mtd_debug`는 `mtd-utils` 패키지에 있습니다.

원본 이미지에는 없었습니다.

```text
flashcp      MISSING
flash_erase  MISSING
mtd-utils    not installed
```

### 5.4 `write_uboot_platform_mtd()`

`armbian-install`은 SPI/MTD에 U-Boot을 쓸 때 다음 함수를 호출합니다.

```text
write_uboot_platform_mtd "$DIR" "/dev/<first-entry>" "$LOG" "$INSTALL_MTD_LIST"
```

함수가 하는 일:

1. `/proc/mtd`를 읽어 파티션 이름과 장치를 매핑합니다.
2. `bootinfo`, `fsbl`, `env`, `esos`, `opensbi`, `uboot`를 순서대로 찾습니다.
3. `flashcp`로 씁니다.
4. `flashcp`가 read-back 검증을 합니다.

> 1차 구현에서는 쓰기 전에 `/var/backups`로 자동 백업했습니다. 최종 코드는 보드 layout이 확인됐으므로 자동 백업을 하지 않습니다. 쓰기 전에 14.13의 수동 백업을 사용하세요.

주의(기여 시): 4번째 인자 `$INSTALL_MTD_LIST`는 이 구현에서 사용하지 않습니다.
이 보드는 NOR 전체가 `mtd0`이고 실제 파티션이 `mtd1..6`이므로, 이름으로 찾아 6개를 모두 씁니다.
upstream 리뷰에서는 이 동작을 계약에 맞춰 정리하라는 지적이 나올 수 있습니다.
정리 방향은 `AIBOX-K3-GUIDE.md` 9.2를 보세요.

#### 왜 `mtdN` 번호를 쓰지 않는가

K1 보드와 K3 보드의 layout이 다릅니다.

K1:

```text
mtd0 bootinfo
mtd1 private
mtd2 fsbl
mtd3 env
mtd4 opensbi
mtd5 uboot
```

K3:

```text
mtd0 d420c000.spi (master)
mtd1 bootinfo
mtd2 fsbl
mtd3 env
mtd4 esos
mtd5 opensbi
mtd6 uboot
```

K1에는 `esos`가 없고, K3에는 있습니다. 또한 `CONFIG_MTD_PARTITIONED_MASTER`가 켜지면 번호가 하나씩 밀립니다. 그래서 **이름**으로 찾아야 안전합니다.

#### 왜 `flashcp`인가

- SPI NOR은 쓰기 전에 erase가 필요합니다.
- `flashcp`는 erase, write, verify를 한 번에 합니다.
- `/dev/mtdN` char device를 사용합니다.
- `dd`로 `/dev/mtdblockN`에 쓰는 방법도 있지만, `flashcp`가 더 명확하고 검증됩니다.

#### 실패하면

- 각 단계에서 `return 1`합니다.
- 로그에 “do not reboot”를 남깁니다.
- 백업이 있으므로 복구할 수 있습니다.

> 위 설명은 1차 구현의 설계입니다. 최종 코드는 보드 layout이 확인됐으므로 백업/예외 처리를 제거하고 `flashcp`만 사용합니다. 최종 함수는 5.6을 보세요.

### 5.5 `run_host_command_logged`의 괄호 문제

처음 작성한 줄:

```bash
run_host_command_logged echo "extraargs=... mtdparts=...:128K@0(bootinfo),..." >> "${SDCARD}"/boot/armbianEnv.txt
```

이 줄은 빌드 중에 실패했습니다.

```text
ERROR: Error 2 occurred in main shell
command failed with error code 2
```

이유:

- `run_host_command_logged`는 명령을 `bash -c "$*"`로 실행합니다.
- `mtdparts` 문자열의 `(`와 `)`가 shell 문법으로 해석됩니다.
- 그래서 “unexpected token” 오류가 납니다.

수정한 줄:

```bash
run_host_command_logged "echo 'extraargs=... mtdparts=...:128K@0(bootinfo),...' >> '${SDCARD}/boot/armbianEnv.txt'"
```

- 전체를 하나의 문자열로 넘깁니다.
- `extraargs` 값은 작은따옴표로 감쌉니다.
- 리다이렉션도 문자열 안에 넣습니다.

이렇게 하면 `bash -c`가 괄호를 일반 문자로 처리합니다.

### 5.6 최종 보드 설정 전체

```bash
# SpacemiT K3 16-core RISC-V SoC, UFS storage

BOARD_NAME="AIBOX-K3"
BOARD_VENDOR="firefly"
BOARDFAMILY="spacemit-k3"

# The 6.18 vendor BSP is the only supported kernel branch.
KERNEL_TARGET="legacy"
KERNEL_TEST_TARGET="legacy"

BOOT_FDT_FILE="spacemit/k3-firefly-aibox-k3.dtb"
BOOTENV_FILE="spacemit.txt"
BOOTSCRIPT="boot-spacemit.cmd:boot.cmd"

# mtd-utils provides flashcp for SPI NOR updates.
PACKAGE_LIST_BOARD="mtd-utils"

IMAGE_PARTITION_TABLE="gpt"
SECTOR_SIZE="4096"
OFFSET="12"

BOOTFS_TYPE="fat"
BOOTSIZE="256"

# SPI NOR layout shared by U-Boot and Linux.
K3_MTDPARTS="d420c000.spi:128K@0(bootinfo),512K@128K(fsbl),64K@640K(env),1M@704K(esos),384K@1728K(opensbi),-@2112K(uboot)"

function post_config_uboot_target__firefly_aibox_k3() {
	write_uboot_platform() {
		echo "U-Boot lives in SPI NOR; not writing it to UFS."
	}
}

function pre_install_kernel_debs__extra_boot_args() {
	display_alert "$BOARD" "Add extra boot arguments" "info"
	run_host_command_logged "echo 'extraargs=net.ifnames=0 mtdparts=${K3_MTDPARTS}' >> '${SDCARD}/boot/armbianEnv.txt'"
}

# Called by armbian-install as:
#   write_uboot_platform_mtd <dir> <device> <log> <mtd_list>
# K3 exposes the whole NOR as mtd0, so resolve partitions by name.
function write_uboot_platform_mtd() {
	local dir="${1}" log="${3:-/dev/stderr}"
	local name file dev

	for name in bootinfo fsbl env esos opensbi uboot; do
		dev="$(awk -v name="\"${name}\"" '$NF == name { sub(":", "", $1); print "/dev/" $1 }' /proc/mtd)"
		case "${name}" in
			bootinfo) file="bootinfo_spinor.bin" ;;
			fsbl)     file="FSBL.bin" ;;
			env)      file="u-boot-env-default.bin" ;;
			esos)     file="esos.itb" ;;
			opensbi)  file="fw_dynamic.itb" ;;
			uboot)    file="u-boot.itb" ;;
		esac
		flashcp -v "${dir}/${file}" "${dev}" >> "${log}" 2>&1 || return 1
	done
	sync
}

function pre_umount_final_image__firefly_aibox_k3_boot() {
	local boot_dir="${MOUNT}/boot"

	cat > "${boot_dir}/env_k3.txt" <<-'EOF'
	console=ttyS0,115200
	bootdelay=5
	knl_name=Image
	dtb_dir=dtb/spacemit
	loglevel=8
	EOF
}
```

---

## 6. 빌드

### 6.1 빌드 호스트

RISC-V 보드에서 직접 빌드하지 않습니다. x86_64 Ubuntu 호스트에서 cross-compile합니다.

- Ubuntu 24.04/26.04 계열
- Docker 또는 Podman
- 16코어 이상, 16GB RAM 이상 권장
- 20GB 이상 여유 공간

### 6.2 요구사항 설치

처음 한 번만:

```bash
cd ~/code/armbian/build
sudo ./compile.sh requirements
```

### 6.3 빌드 명령

```bash
./compile.sh build BOARD=firefly-aibox-k3 BRANCH=legacy \
  BUILD_DESKTOP=no BUILD_MINIMAL=no EXPERT=yes \
  KERNEL_CONFIGURE=no RELEASE=resolute
```

| 옵션 | 의미 |
|---|---|
| `BOARD=firefly-aibox-k3` | AIBOX-K3 보드 설정 |
| `BRANCH=legacy` | SpacemiT 6.18 vendor kernel |
| `BUILD_DESKTOP=no` | 데스크톱 제외 |
| `BUILD_MINIMAL=no` | 최소 이미지 아님 |
| `EXPERT=yes` | expert 옵션 |
| `KERNEL_CONFIGURE=no` | kernel config를 묻지 않음 |
| `RELEASE=resolute` | Ubuntu 26.04 기반 |

캐시가 있으면 약 4~5분 걸립니다. 처음 빌드는 더 오래 걸립니다.

### 6.4 결과 파일

```text
output/images/Armbian-unofficial_26.11.0-trunk_Firefly-aibox-k3_resolute_legacy_6.18.3.img
output/images/Armbian-unofficial_..._6.18.3.img.sha
output/images/Armbian-unofficial_..._6.18.3.img.txt

output/debs/linux-u-boot-firefly-aibox-k3-legacy_26.11.0-trunk_riscv64__2022.10-Se471-P78cf-Hde31-V0a62-B56a7-R448a.deb
```

이 문서 작성 시 이미지 SHA256:

```text
6def878d1a589e451a360e333c1e08dbf12ecc8f045f8ba58d6eb9f271991529
```

`*.img`는 UFS 전체에 쓸 수 있는 디스크 이미지입니다. SPI NOR 부트로더는 이미지 안에 들어 있지만, 이미지를 UFS에 썼다고 SPI NOR이 자동으로 바뀌지는 않습니다.

`compile.sh`는 TitanTools용 zip을 자동으로 만들지 않습니다. zip이 필요하면 **17.3**의 조립 절차를 실행하세요. Armbian installer만 쓸 경우 zip은 필요 없습니다.

### 6.5 빌드 후 확인

이미지 확인:

```bash
IMG=$(ls -t output/images/*.img | head -1)
cat "${IMG}.sha"
sudo losetup -b 4096 -fP --show "$IMG"
```

U-Boot deb 확인:

```bash
DEB=$(ls -t output/debs/linux-u-boot-firefly-aibox-k3-legacy_*.deb | head -1)
dpkg-deb -x "$DEB" /tmp/uboot-check
grep -n write_uboot_platform_mtd /tmp/uboot-check/usr/lib/u-boot/platform_install.sh
```

`write_uboot_platform_mtd()`가 보여야 합니다. 이 함수가 SPI NOR 6개 파티션을 이름으로 찾아 씁니다.

### 6.6 이미지에 들어 있는 것

```text
boot.scr
armbianEnv.txt (fdtfile + mtdparts)
Image
uInitrd
dtb/spacemit/k3-firefly-aibox-k3.dtb
mtd-utils / flashcp
linux-u-boot-firefly-aibox-k3-legacy
write_uboot_platform_mtd()
thermal_zone0 온도 수정
```

즉 이 이미지는 **UFS용 rootfs + SPI NOR용 U-Boot deb + distro boot 설정**을 모두 포함합니다.

---

## 7. 설치 방식 선택

| 방법 | 장점 | 제약 | 사용할 때 |
|---|---|---|---|
| **Armbian installer** | Armbian 표준 도구, UFS+SPI NOR 자동 | live USB 부팅 필요 | SPI NOR에 Armbian U-Boot가 있을 때 |
| **TitanTools** | 현재 보드에서 검증됨, 첫 설치 가능 | Windows 도구/JSON 필요 | 첫 설치, 복구, 클린 설치 |
| **수동 flashcp** | 도구 최소 | 명령을 직접 실행 | 설치된 Linux에서 U-Boot만 교체 |
| **fastboot** | SPI NOR만 직접 교체 | 시리얼/타이밍 주의 | 복구, 부트로더만 갱신 |

### 7.1 Armbian installer

자세한 절차는 **19장**을 보세요.

핵심:

```bash
# live USB로 부팅한 뒤
lsblk -o NAME,SIZE,TYPE,MODEL,MOUNTPOINTS
sudo armbian-install --target /dev/sda --boot mtd --fs ext4 --yes
```

- `--target /dev/sda`: UFS
- `--boot mtd`: SPI NOR에 U-Boot
- 현재 실행 중인 디스크는 target이 될 수 없습니다. live USB가 필요합니다.

### 7.2 TitanTools

자세한 절차는 **17장**을 보세요.

```text
UFS:      partition_universal.json + bootfs.img + rootfs.ext4
SPI NOR:  partition_4M.json + U-Boot 파일 6개
```

### 7.3 수동

이미 설치된 Armbian에서 U-Boot만 교체:

```bash
sudo dpkg -i linux-u-boot-firefly-aibox-k3-legacy_*.deb
sudo bash -c '
  source /usr/lib/u-boot/platform_install.sh
  write_uboot_platform_mtd "$DIR" /dev/mtd0 /tmp/spinor.log
'
```

`write_uboot_platform_mtd()`는 자동 백업을 하지 않습니다. 쓰기 전에 **14.13**의 수동 백업을 실행하세요.

---

## 8. 검증 결과 요약

최종 빌드 `2022.10-Se471-P78cf-Hde31-V0a62-B56a7-R448a` 기준:

```text
U-Boot SPL 2022.10_armbian-2022.10-Se471-P78cf-Hde31-V0a62-B56a7-R448a
U-Boot 2022.10_armbian-2022.10-Se471-P78cf-Hde31-V0a62-B56a7-R448a
Loading K3 environment
Loading Armbian environment (scsi 0:1)
Booting Armbian (scsi 0:1)
Starting kernel ...
```

SPI NOR 6개 파티션:

```text
bootinfo  MATCH
fsbl      MATCH
env       MATCH
esos      MATCH
opensbi   MATCH
uboot     MATCH
```

시스템:

```text
MPXY:     mailbox registered with 5 channels
팬:       ctf2301_fan, PWM 60/255, 1,444 RPM
CPU 온도: thermal_zone0 39°C
failed:   0 loaded units
```

다른 K3 보드 영향 없음:

```text
k3picoitx
  BOOTSOURCE   = spacemit-com/uboot-2022.10
  BOOTBRANCH   = tag:k3-br-v1.0.8
  BOOTPATCHDIR = legacy/u-boot-spacemit-k3
  ESOS         = 450,419 bytes
```

---

## 9. 문제 해결

### 9.1 `/proc/mtd`가 비어 있다

원인:

- `mtdparts`가 kernel command line에 없다.
- `mtdparts`의 장치 이름이 실제 SPI NOR 이름과 다르다.
- `CONFIG_MTD_CMDLINE_PARTS`가 꺼져 있다.

확인:

```bash
cat /proc/cmdline
dmesg | grep -iE 'mtd|spi'
cat /sys/class/mtd/mtd*/name
```

해결:

- `extraargs`에 `mtdparts=d420c000.spi:...`가 있는지 확인합니다.
- 실제 장치 이름이 `d420c000.spi`인지 확인합니다.
- 다르면 이름을 맞춥니다.

### 9.2 `flashcp: not found`

원인: `mtd-utils`가 설치되지 않음.

해결:

```bash
sudo apt install -y mtd-utils
```

새 이미지에는 `PACKAGE_LIST_BOARD="mtd-utils"`로 포함됩니다.

### 9.3 `/dev/mtdN: Permission denied`

원인: 일반 사용자 권한.

해결:

```bash
sudo ...
```

또는 `sudo -i`로 root가 됩니다.

### 9.4 `/boot/boot.scr`이 없다

원인:

- `BOOTSCRIPT` 설정이 빠졌다.
- 이미지가 새로 빌드되지 않았다.
- bootfs가 잘못 쓰였다.

해결:

- `config/boards/firefly-aibox-k3.wip`에 `BOOTSCRIPT`가 있는지 확인합니다.
- 새 이미지를 빌드합니다.
- `/boot`를 마운트해서 `boot.scr`을 확인합니다.

**이 상태에서 SPI NOR을 쓰면 부팅이 깨집니다.**

### 9.5 Armbian U-Boot가 이미지를 못 찾는다

증상:

```text
Scanning for devices...
...
########### boot failed ...
```

확인:

- `/boot.scr`이 bootfs root에 있는가?
- `bootfs` 파티션이 1번인가?
- UFS가 `scsi 0`으로 보이는가?
- `armbianEnv.txt`의 `fdtfile`이 맞는가?

시리얼 로그에서 `Loading Armbian environment (scsi 0:1)`가 나오는지 확인합니다.

### 9.6 UFS/SCSI가 안 보인다

AIBOX-K3는 UFS를 SCSI로 노출합니다. U-Boot의 `board_scsi_scan_once()`가 초기화합니다. `scsi 0`이 안 보이면 시리얼 로그에서 `ufshcd`/`scsi` 메시지를 확인합니다.

### 9.7 `overlay_prefix=k1`

`config/bootenv/spacemit.txt`는 K1/K3 공용이라 `overlay_prefix=k1`이 들어갑니다. 기본 부팅에는 영향이 없습니다. K3 overlay를 쓸 때는 `OVERLAY_PREFIX="k3"`를 보드 설정에 추가하거나 K3 전용 bootenv 파일을 만드는 것이 좋습니다.

### 9.8 커널 에러 메시지

다음은 이번 검증에서 보인 메시지입니다.

```text
Spacemit K3: cpu0 OPP init failed (-517)
pcieport ... of_irq_parse_pci: failed with rc=-22
r8152 ... Invalid ether addr 00:00:00:00:00:00
```

이들은 SPI NOR/U-Boot 변경과 무관한 기존 SoC/드라이버 특성으로 보입니다. 부팅과 네트워크는 정상입니다.

### 9.9 `erasespi`를 쓰지 말 것

`erasespi`는 4MiB만 지웁니다. 전체 8MiB NOR을 지우는 명령이 아닙니다. 복구 계획 없이 사용하지 마세요.

### 9.10 하드웨어/펌웨어 사실 확인 명령

"이 보드가 어떤 칩을 쓰는지", "이 파일이 진짜 무엇인지"를 확인하는 명령입니다.

```bash
# 1) 저장 장치와 파티션
lsblk -o NAME,SIZE,TYPE,MODEL,TRAN      # UFS가 /dev/sda 인지
cat /proc/mtd                           # SPI NOR 6개 파티션 이름/크기
cat /proc/cmdline | tr ' ' '\n' | grep mtdparts

# 2) 부트 체인
cat /boot/armbianEnv.txt                # fdtfile, extraargs(mtdparts)
ls -l /boot                             # Image, uInitrd, dtb, boot.scr, env_k3.txt

# 3) hwmon / thermal (Fan controller vs SoC sensor)
cat /sys/class/hwmon/hwmon*/name        # ctf2301 -> hwmon0 은 팬 컨트롤러
for c in /sys/class/thermal/cooling_device*; do echo "$c $(cat $c/type)"; done
for z in /sys/class/thermal/thermal_zone*; do echo "$z $(cat $z/type) $(cat $z/temp)"; done

# 4) 펌웨어 파일 정체
strings -n 6 esos.itb | head -3         # "ESOS dual-core image for K3"
strings -n 8 fw_dynamic.itb | head -2   # "OpenSBI fw_dynamic Firmware"
strings -n 8 u-boot.itb | grep -i FIT   # "U-boot FIT image for k3"
od -A d -t x1 bootinfo_spinor.bin       # 80바이트, offset 8에 "NORF" 매직
```

해석:

- `hwmon0` 이름이 `ctf2301`이면 그 hwmon은 SoC 온도 센서가 아니라 **팬 컨트롤러**입니다.
  CPU 온도는 `thermal_zone0/temp`입니다.
- `cooling_device0`의 `type`이 `ctf2301_fan`이면 그 디바이스가 PWM 제어를 받습니다.
- `esos.itb`의 "dual-core image"와 "Using data interaction with AP"는 ESOS가
  보조(RT) 코어에서 돌면서 Linux와 mailbox로 통신한다는 뜻입니다.

---

## 10. 커밋 메시지와 PR 본문

### 10.1 커밋 메시지

```text
board: firefly-aibox-k3: add SPI NOR and distro boot support

The AIBOX-K3 boots from SPI NOR and stores the OS on UFS. Add
boot.scr/armbianEnv, mtdparts, mtd-utils, and
write_uboot_platform_mtd() so Linux can update the SPI NOR bootloader.

Use the Firefly k3/linux6.18_release_v1.0.0a tag and its 506,499-byte
ESOS blob only for this board. Keep other spacemit-k3 boards on the
family defaults. Rebase U-Boot 001-003 for distro boot.

Tested on AIBOX-K3 (8GB/128GB, UFS):
- boot via boot.scr and armbianEnv.txt
- all six SPI NOR partitions match the build artifacts
- MPXY, fan, and CPU temperature normal
- k3picoitx build unchanged
- recovery via Boot button + USB fastboot
```

### 10.2 PR 제목

```text
board: firefly-aibox-k3: add SPI NOR and distro boot support
```

### 10.3 최종 PR 본문 (3차 수정 포함)

````markdown
## Summary

This series makes the Firefly AIBOX-K3 boot from SPI NOR with an
Armbian-built U-Boot, and fixes the fan and CPU temperature reporting
that broke with the older SpacemiT firmware tag.

The initial board support only wrote bootfs/rootfs to UFS and kept the
factory U-Boot. It could not boot with the Armbian U-Boot because the
image had no `boot.scr`, no `mtdparts` in the Armbian boot path, and no
`write_uboot_platform_mtd()`.

While testing the SPI NOR update, the CTF2301 fan controller stopped
probing. The root cause was a firmware source mismatch: Armbian used the
older `spacemit-com` `k3-br-v1.0.8` tag, while the factory firmware uses
the Firefly `k3/linux6.18_release_v1.0.0a` tag. That mismatch breaks the
SBI MPXY extension, which in turn keeps `i2c6` and the CTF2301 fan
controller from probing.

## Changes

### 1. board: firefly-aibox-k3: add SPI NOR bootloader update

- Add `BOOTSCRIPT` and `BOOTENV_FILE` so the image contains `boot.scr`
  and `armbianEnv.txt` with `fdtfile`.
- Add `mtdparts` to `extraargs` so Linux exposes the SPI NOR partitions.
- Add `mtd-utils` for `flashcp`.
- Add `write_uboot_platform_mtd()`:
  - finds partitions by name, not by `mtdN` number
  - writes with `flashcp` (erase + write + verify)

### 2. board: firefly-aibox-k3: use Firefly firmware tag

- Switch U-Boot and OpenSBI to
  `Firefly-spacemit-linux` tag `k3/linux6.18_release_v1.0.0a` in a
  board-level `post_family_config` hook. The family defaults stay on
  `spacemit-com` `k3-br-v1.0.8` for other K3 boards.
- Add the ESOS blob under the board-specific path
  `packages/blobs/riscv64/spacemit-k3/aibox-k3/esos.itb` (506,499
  bytes). The family default blob stays unchanged.
- Add `legacy/u-boot-spacemit-k3-firefly` with patches 001-006.
- Rebase 001-003 onto the Firefly tag:
  - 001: rename boot variables for distro boot
  - 002: enable distro boot defconfig options
  - 003: replace the vendor env with the Armbian `autoboot`/`bootfunc`
    env
- Add an empty `atf-spacemit-k3-firefly` patch dir so the family ATF
  patch is not applied to Firefly OpenSBI.
- The image now boots through `/boot.scr` and `armbianEnv.txt`.

### 3. bsp: firefly-aibox-k3: use SoC thermal zone for CPU temp

- The CTF2301 fan controller registers as `hwmon0` on K3. Its
  `temp1_input` is not the SoC temperature and uses a different scale.
- Use `thermal_zone0/temp` for `soctemp` when `${BOARD}` is
  `firefly-aibox-k3`.

## Why

The factory U-Boot env uses `nor_boot` -> `set_nor_args` -> `boot_kernel`
and reads `env_k3.txt`. The Armbian U-Boot env uses `autoboot` ->
`bootfunc` -> `/boot.scr`. The first Firefly build kept the vendor env,
so it did not use the Armbian boot path. Rebasing 001-003 switches the
env to `autoboot`/`bootfunc` and makes `boot.scr` the real boot path.

The factory U-Boot appends `mtdparts` to `bootargs`. The Armbian
`boot.scr` does not. Without `mtdparts`, Linux does not create
`/dev/mtd1..6`.

`CONFIG_MTD_PARTITIONED_MASTER=y` exposes the whole flash as `mtd0`, so
the partitions are `mtd1..6`. The numbering also differs from K1 boards.
Using names avoids hard-coded offsets.

The MPXY failure was visible in `dmesg`:

```text
riscv-sbi-mpxy-mbox soc:mpxy_mbox@0: SBI MPXY extension not available
```

With the Firefly firmware tag:

```text
riscv-sbi-mpxy-mbox soc:mpxy_mbox@0: mailbox registered with 5 channels
```

The CTF2301 fan controller then probes on `i2c6`, and the fan is
controlled again.

## Testing

Hardware: Firefly AIBOX-K3, 8GB RAM, 128GB UFS, kernel 6.18.3 legacy.

Build:

```bash
./compile.sh uboot BOARD=firefly-aibox-k3 BRANCH=legacy
```

U-Boot package:

```text
linux-u-boot-firefly-aibox-k3-legacy_26.11.0-trunk_riscv64__2022.10-Se471-P78cf-Hde31-V0a62-B56a7-R448a.deb
```

Test sequence:

1. Wrote bootfs/rootfs to UFS, kept factory SPI NOR.
2. Booted the new image with the factory U-Boot. Verified:
   - `/boot/boot.scr`
   - `/boot/armbianEnv.txt` with `fdtfile` and `mtdparts`
   - `/proc/mtd` with 6 partitions
   - `mtd-utils` and `write_uboot_platform_mtd()`
3. Backed up the factory SPI NOR.
4. Wrote the Firefly firmware set (including the rebased 001-003
   patches) to SPI NOR with `write_uboot_platform_mtd()`.
5. Rebooted and verified the distro boot path:

```text
U-Boot SPL 2022.10_armbian-2022.10-Se471-P78cf-Hde31-V0a62-B56a7-R448a
U-Boot 2022.10_armbian-2022.10-Se471-P78cf-Hde31-V0a62-B56a7-R448a
Loading K3 environment
Loading Armbian environment (scsi 0:1)
Booting Armbian (scsi 0:1)
Starting kernel ...
```

6. Verified fan, temperature, and MPXY:

```text
mailbox registered with 5 channels
/sys/class/thermal/cooling_device0 type=ctf2301_fan cur=60 max=255
fan1_input=1444
pwm1=60
thermal_zone0=39000
CPU temp: 39°C
```

7. All six SPI NOR partitions match the build artifacts:

```text
bootinfo  MATCH
fsbl      MATCH
env       MATCH
esos      MATCH
opensbi   MATCH
uboot     MATCH
```

8. `systemctl --failed`: 0 loaded units.
9. Verified `k3picoitx` still builds with `spacemit-com`
   `k3-br-v1.0.8`, `legacy/u-boot-spacemit-k3`, and the 450,419-byte
   ESOS blob.

## Recovery

- Boot button + USB-C + TitanTools/fastboot.
- Factory SPI NOR backup taken before writing.
- `spi-flash-howto.txt` in the U-Boot package documents the fastboot
  sequence.

## Follow-ups

- The CTF2301 hwmon temperature scale is 10x. The BSP now uses
  `thermal_zone0`, so the display is correct; a driver fix can follow.
- The board remains `.wip`.
````

---

## 11. 부록

### 11.1 SPI NOR 파티션 요약

| mtd | 이름 | 크기 | offset | 내용 |
|---|---|---:|---:|---|
| mtd0 | `d420c000.spi` | 8MiB | 0 | master |
| mtd1 | `bootinfo` | 128KiB | 0 | BootROM용 FSBL 정보 |
| mtd2 | `fsbl` | 512KiB | 128KiB | U-Boot SPL |
| mtd3 | `env` | 64KiB | 640KiB | U-Boot env |
| mtd4 | `esos` | 1MiB | 704KiB | ESOS |
| mtd5 | `opensbi` | 384KiB | 1728KiB | OpenSBI |
| mtd6 | `uboot` | 5.9MiB | 2112KiB | U-Boot proper |

### 11.2 UFS 파티션 요약

| 파티션 | 장치 | 크기 | 파일시스템 | 이름 | 마운트 |
|---|---|---:|---|---|---|
| 1 | `/dev/sda1` | 256MiB | FAT | `bootfs` | `/boot` |
| 2 | `/dev/sda2` | 119GB | ext4 | `rootfs` | `/` |

### 11.3 주요 명령 모음

#### 상태 확인

```bash
uname -a
cat /proc/cmdline
cat /proc/mtd
for d in /sys/class/mtd/mtd*; do [ -f "$d/name" ] && echo "$d $(cat "$d/name") $(cat "$d/size")"; done
lsblk -o NAME,SIZE,TYPE,FSTYPE,LABEL,PARTLABEL,MOUNTPOINTS
```

#### SPI NOR 버전 확인

```bash
sudo dd if=/dev/mtd6 bs=1M count=4 status=none | strings | grep -E 'U-Boot 20|Armbian'
```

#### SPI NOR env 확인

```bash
sudo dd if=/dev/mtd3 bs=64k status=none | strings | grep -E '^(bootcmd|autoboot|bootfunc|mtdparts)='
```

#### SPI NOR 해시 비교

```bash
D=/usr/lib/linux-u-boot-legacy-firefly-aibox-k3
for pair in "bootinfo bootinfo_spinor.bin" "fsbl FSBL.bin" "env u-boot-env-default.bin" "esos esos.itb" "opensbi fw_dynamic.itb" "uboot u-boot.itb"; do
  set -- $pair; name=$1; file=$2
  dev=""
  for s in /sys/class/mtd/mtd*; do
    [ -f "$s/name" ] || continue
    [ "$(cat "$s/name")" = "$name" ] && dev="/dev/${s##*/}" && break
  done
  size=$(stat -c %s "$D/$file")
  cur=$(sudo dd if="$dev" bs=4096 count=$(( (size + 4095) / 4096 )) status=none | head -c "$size" | sha256sum | cut -d' ' -f1)
  want=$(sha256sum "$D/$file" | cut -d' ' -f1)
  printf '%-8s %s\n' "$name" "$([ "$cur" = "$want" ] && echo MATCH || echo DIFF)"
done
```

#### SPI NOR 쓰기

```bash
sudo bash -c '
  set -e
  source /usr/lib/u-boot/platform_install.sh
  write_uboot_platform_mtd "$DIR" /dev/mtd0 /tmp/spinor.log
'
```

#### 수동 복구

```bash
sudo flashcp -v bootinfo.bin /dev/mtd1
sudo flashcp -v fsbl.bin     /dev/mtd2
sudo flashcp -v env.bin      /dev/mtd3
sudo flashcp -v esos.bin     /dev/mtd4
sudo flashcp -v opensbi.bin  /dev/mtd5
sudo flashcp -v uboot.bin    /dev/mtd6
```

### 11.4 용어집

| 용어 | 설명 |
|---|---|
| SoC | System on Chip. CPU, GPU, 주변장치를 한 칩에 모은 것 |
| UFS | Universal Flash Storage. 스마트폰/임베디드용 고속 저장 장치 |
| SPI NOR | SPI로 연결된 NOR 플래시. 부트로더 저장에 주로 사용 |
| BootROM | SoC에 구워진 첫 단계 부팅 코드 |
| FSBL | First Stage Boot Loader. U-Boot SPL에 해당 |
| SPL | Secondary Program Loader. DRAM 초기화 담당 |
| OpenSBI | RISC-V M-mode 런타임 |
| ESOS | SpacemiT의 전력/실시간 코어용 펌웨어 |
| U-Boot | 운영체제를 로드하는 부트로더 |
| boot.scr | U-Boot이 실행하는 스크립트 |
| extlinux | U-Boot이 읽는 부팅 설정 파일 형식 |
| DTB | Device Tree Blob. 하드웨어 설명 파일 |
| MTD | Memory Technology Device. 플래시 메모리 추상화 |
| mtdparts | Linux에 MTD 파티션을 알려주는 kernel command line |
| GPT | GUID Partition Table |
| UFS bootfs | 커널/initrd/DTB가 있는 FAT 파티션 |
| UFS rootfs | Linux 루트 파일시스템 |
| fastboot | Android 계열 USB 플래싱 프로토콜 |
| TitanTools | Firefly의 USB 플래싱 도구 |
| `flashcp` | MTD 파티션에 erase/write/verify하는 도구 |
| `mtd-utils` | `flashcp`, `flash_erase` 등을 제공하는 패키지 |

### 11.5 참고 자료

- Armbian build framework: <https://github.com/armbian/build>
- Firefly AIBOX-K3 wiki: <https://wiki.t-firefly.com/en/AIBOX-K3/>
- Firefly AIBOX-K3 serial debug: <https://community.t-firefly.com/en/docs/products/computers/AIBOX-K3/debug>
- Firefly firmware upgrade: <https://wiki.t-firefly.com/en/AIBOX-K3/upgrade_firmware.html>
- SpacemiT K3 U-Boot: <https://github.com/spacemit-com/uboot-2022.10>
- SpacemiT K3 OpenSBI: <https://github.com/spacemit-com/opensbi>
- SpacemiT K3 Linux: <https://github.com/spacemit-com/linux-6.18>
- K3 boot flow 문서: <https://github.com/ganboing/K3-Docs>
- Armbian board config 문서: `config/boards/README.md`

---

## 12. 펌웨어 소스 선택 근거 (팬 문제 이력)

> 이 장은 **왜 AIBOX-K3만 Firefly 펌웨어 소스를 쓰는가**를 설명하는 근거 자료입니다.
> contrib 관점에서는 `AIBOX-K3-GUIDE.md` 3장만 보면 됩니다.
> 여기의 문제(MPXY, CTF2301, 온도)는 **증상**일 뿐이고, 실제 원인은 "Armbian이 쓰던 U-Boot/OpenSBI
> 소스가 공장 펌웨어와 달랐던 것"입니다.

### 12.1 증상

SPI NOR에 Armbian U-Boot를 쓴 뒤 다음 증상이 나타났습니다.

- 팬이 계속 최대 속도로 회전
- CPU 온도가 welcome 메시지에서 `335°C`로 표시
- 실제 SoC 온도는 `thermal_zone0` 기준 30~35°C
- `systemctl --failed`는 0

### 12.2 원인 1: MPXY 실패로 팬 제어 불가

팬은 `CTF2301` 칩이 `i2c6`에서 제어합니다.

```text
/soc/i2c@d4018800/ctf2301@4c
compatible = "sensylink,ctf2301"
```

문제가 있을 때의 로그:

```text
riscv-sbi-mpxy-mbox soc:mpxy_mbox@0: SBI MPXY extension not available
platform soc:power-management@0:power-controller: deferred probe pending:
  platform: supplier soc:mpxy_mbox@0 not ready
platform d4018800.i2c: deferred probe pending:
  platform: supplier soc:power-management@0:power-controller not ready
```

연쇄:

```text
mpxy_mbox 실패
  -> power-controller defer
    -> i2c6 defer
      -> ctf2301 probe 안 됨
        -> 팬 기본값(최대 속도)
```

공장 펌웨어에서는 다음 로그가 나왔습니다.

```text
riscv-sbi-mpxy-mbox soc:mpxy_mbox@0: mailbox registered with 5 channels
```

즉 공장 펌웨어에서는 MPXY가 정상입니다.

### 12.3 원인 2: Armbian과 공장 펌웨어의 소스가 다름

공장 SDK manifest:

```text
k3/linux6.18_release_v1.0.0a
```

| 구성 | 공장 | Armbian (수정 전) |
|---|---|---|
| U-Boot | `Firefly-spacemit-linux/uboot-2022.10` `e471ad131b` | `spacemit-com/uboot-2022.10` `3243c5fc` |
| OpenSBI | `Firefly-spacemit-linux/opensbi` `842d995` | `spacemit-com/opensbi` `1598ca8` |
| ESOS | SDK `esos.itb` 506,499 bytes | blobs `esos.itb` 450,419 bytes |

공장 U-Boot 커밋:

```text
e471ad131b riscv: dts: update k3-firefly-common from k3_com260_ifx
```

Armbian은 더 오래된 `spacemit-com` `k3-br-v1.0.8`을 사용하고 있었습니다. 이 조합에서는 RPMI/MPXY 초기화가 실패합니다.

### 12.4 원인 3: 온도 표시가 CTF2301 hwmon을 읽음

`CTF2301`이 probe되면 `hwmon0`으로 등록됩니다.

```text
/sys/class/hwmon/hwmon0/name = ctf2301
/sys/class/hwmon/hwmon0/temp1_input = 345625
/sys/class/hwmon/hwmon0/temp2_input = 363750
```

`armbian-hardware-monitor`는 `hwmon0/temp1_input`을 `soctemp`로 사용합니다.

```text
/etc/armbianmonitor/datasources/soctemp
  -> /sys/class/hwmon/hwmon0/temp1_input
```

그래서 345,625를 345°C로 표시했습니다. 실제 SoC 온도는 `thermal_zone0`입니다.

```text
/sys/class/thermal/thermal_zone0/temp = 35000
```

`CTF2301`의 hwmon 온도는 팬 컨트롤러 자체 센서이고, 스케일도 SoC thermal zone과 다릅니다. CPU 온도 표시에는 `thermal_zone0`을 써야 합니다.

### 12.5 최종 수정

#### 1) AIBOX-K3에서만 Firefly 태그 사용

family 기본값은 `spacemit-com` `k3-br-v1.0.8` 그대로 둡니다. AIBOX-K3 보드 hook에서만 Firefly 태그로 바꿉니다.

`config/boards/firefly-aibox-k3.wip`:

```bash
function post_family_config__firefly_aibox_k3_use_firefly_firmware() {
	display_alert "$BOARD" "Use Firefly firmware tag and distro boot" "info"

	declare -g BOOTTAG="k3/linux6.18_release_v1.0.0a"
	declare -g ATFSOURCE="https://github.com/Firefly-spacemit-linux/opensbi.git"
	declare -g ATFBRANCH="tag:${BOOTTAG}"
	declare -g ATFPATCHDIR="atf-spacemit-k3-firefly"
	declare -g BOOTSOURCE="https://github.com/Firefly-spacemit-linux/uboot-2022.10.git"
	declare -g BOOTBRANCH="tag:${BOOTTAG}"
	declare -g BOOTPATCHDIR="legacy/u-boot-spacemit-k3-firefly"
	declare -g K3_ESOS_BLOB="${SRC}/packages/blobs/riscv64/spacemit-k3/aibox-k3/esos.itb"
}
```

`ATFPATCHDIR="atf-spacemit-k3-firefly"`는 패치가 없는 디렉터리입니다(`.gitkeep`만 있음). Firefly OpenSBI에는 기존 ATF 패치를 적용하지 않습니다.

#### 2) ESOS blob (AIBOX-K3 전용)

family 기본 blob은 그대로 두고, AIBOX-K3용 blob을 따로 둡니다.

```bash
packages/blobs/riscv64/spacemit-k3/esos.itb          # family 기본 (450,419)
packages/blobs/riscv64/spacemit-k3/aibox-k3/esos.itb # AIBOX-K3 (506,499)
```

`uboot_custom_postprocess()`는 한 줄만 바꿉니다.

```diff
+	local esos_blob="${K3_ESOS_BLOB:-${SRC}/packages/blobs/riscv64/spacemit-k3/esos.itb}"
 	if [[ -d "${uboot_work_dir}" ]]; then
 		display_alert "$BOARD" "Adding ESOS blob" "info"
-		cp -fv "${SRC}/packages/blobs/riscv64/spacemit-k3/esos.itb" "${uboot_work_dir}/"
+		cp -fv "${esos_blob}" "${uboot_work_dir}/"
```

이렇게 하면 `K3_ESOS_BLOB`가 없는 다른 K3 보드는 기존 450,419 blob을 그대로 씁니다.

#### 3) U-Boot 패치 디렉터리

새 디렉터리:

```text
patch/u-boot/legacy/u-boot-spacemit-k3-firefly/
  001-SpacemiT-K3-fixup-boot-variables.patch
  002-Update-SpacemiT-K3-defconfig.patch
  003-Update-SpacemiT-K3-Env.patch
  004-board-spacemit-k3-Initialize-miiphy-early-to-fix-EFI.patch
  005-lib-printf-Revert-spacemit-s-buggy-CONFIG_PRINT_TIME.patch
  006-Fixup-circular-deps.patch
```

`001`, `002`, `003`은 원래 `k3-br-v1.0.8` 기준이라 Firefly 태그에 그대로 적용되지 않았습니다. Firefly 태그에서 다시 작성해 아래처럼 동작하게 했습니다.

- `001`: `boot_device`/`boot_devnum`/`bootfs_part`를 `devtype`/`devnum`/`distro_bootpart`로 변경
- `002`: `CONFIG_DISTRO_DEFAULTS`, `CONFIG_BOOTSTD_FULL`, `CONFIG_BOOTCOMMAND="run distro_bootcmd"` 등 distro boot 옵션 추가
- `003`: vendor env를 Armbian `autoboot`/`bootfunc` env로 교체

세 패치 모두 Firefly 태그에 clean apply됩니다.

ATF 패치 `001-k3-fix-rcpu1-reserved-mem-overlap.patch`는 family 기본으로 유지합니다. AIBOX-K3는 Firefly OpenSBI를 쓰므로 빈 `atf-spacemit-k3-firefly` 디렉터리를 지정해 이 패치를 적용하지 않습니다.

#### 4) CPU 온도 표시 수정

`packages/bsp/common/usr/lib/armbian/armbian-hardware-monitor`:

```diff
-	# SoC temp: check sun7i legacy exception first, then standard path(s)
-	if [[ -d "/sys/devices/platform/a20-tp-hwmon" ]]; then
+	# SoC temp: check board-specific exceptions first, then standard path(s)
+	if [[ ${BOARD} == "firefly-aibox-k3" ]]; then
+		# CTF2301 (hwmon0) is the fan controller, not the SoC sensor.
+		ln -fs /sys/devices/virtual/thermal/thermal_zone0/temp /etc/armbianmonitor/datasources/soctemp
+	elif [[ -d "/sys/devices/platform/a20-tp-hwmon" ]]; then
```

적용 후:

```text
/etc/armbianmonitor/datasources/soctemp
  -> /sys/devices/virtual/thermal/thermal_zone0/temp
```

welcome 메시지:

```text
CPU temp: 35°C
```

### 12.6 빌드 결과

```text
linux-u-boot-firefly-aibox-k3-legacy_26.11.0-trunk_riscv64__2022.10-Se471-P78cf-Hde31-V0a62-B56a7-R448a.deb
```

U-Boot version:

```text
U-Boot 2022.10_armbian-2022.10-Se471-P78cf-Hde31-V0a62-B56a7-R448a
```

SPI NOR 파일 크기:

```text
FSBL.bin                 453984
fw_dynamic.itb           271311
esos.itb                 506499
u-boot.itb              2683538
u-boot-env-default.bin    16384
```

`u-boot.itb`는 distro boot 명령이 추가되어 2,683,538 bytes로 커졌습니다. `uboot` 파티션(5.9MiB)에 들어갑니다.

### 12.7 SPI NOR 갱신

`write_uboot_platform_mtd`로 6개 파티션을 한 번에 썼습니다.

```bash
sudo bash -c '
  source /usr/lib/u-boot/platform_install.sh
  write_uboot_platform_mtd "$DIR" /dev/mtd0 /tmp/spinor-firefly.log
'
```

결과:

```text
SPI NOR bootloader written
```

파티션 검증:

```text
bootinfo  MATCH
fsbl      MATCH
env       MATCH
esos      MATCH
opensbi   MATCH
uboot     MATCH
```

### 12.8 재부팅 후 검증

distro boot 시리얼 로그:

```text
U-Boot SPL 2022.10_armbian-2022.10-Se471-P78cf-Hde31-V0a62-B56a7-R448a
U-Boot 2022.10_armbian-2022.10-Se471-P78cf-Hde31-V0a62-B56a7-R448a
Loading K3 environment
Loading Armbian environment (scsi 0:1)
Booting Armbian (scsi 0:1)
Starting kernel ...
```

`/proc/cmdline`도 `boot.scr` 경로로 바뀌었습니다.

```text
console=tty0 console=ttyS0,115200n8 root=PARTUUID=... rootfstype=ext4 rw fsck.repair=yes rootwait loglevel=1 net.ifnames=0 mtdparts=d420c000.spi:...
```

MPXY:

```text
riscv-sbi-mpxy-mbox soc:mpxy_mbox@0: mailbox registered with 5 channels
```

팬:

```text
/sys/class/thermal/cooling_device0 type=ctf2301_fan cur=16 max=255
fan1_input=548
pwm1=16
```

온도가 올라가면 PWM과 RPM도 함께 올라갑니다.

```text
cur=60 max=255
fan1_input=1444
pwm1=60
thermal_zone0=39000
CPU temp: 39°C
```

시스템:

```text
systemctl --failed: 0 loaded units
```

SPI NOR 6개 파티션도 새 `P78cf-Hde31-V0a62` 파일과 다시 비교했습니다.

```text
mtd1 bootinfo  MATCH
mtd2 fsbl      MATCH
mtd3 env       MATCH
mtd4 esos      MATCH
mtd5 opensbi   MATCH
mtd6 uboot     MATCH
```

재부팅 후에도 동일하게 유지됩니다.

### 12.9 최종 커밋 계획

1. **board: firefly-aibox-k3: add SPI NOR bootloader update**
   - `config/boards/firefly-aibox-k3.wip`
   - `BOOTSCRIPT`, `BOOTENV_FILE`, `mtdparts`, `mtd-utils`, `write_uboot_platform_mtd`

2. **board: firefly-aibox-k3: use Firefly firmware and distro boot**
   - `config/boards/firefly-aibox-k3.wip`의 `post_family_config` hook
   - `packages/blobs/riscv64/spacemit-k3/aibox-k3/esos.itb`
   - `patch/u-boot/legacy/u-boot-spacemit-k3-firefly/`
   - `patch/atf/atf-spacemit-k3-firefly/`
   - `config/sources/families/spacemit-k3.conf`의 ESOS 선택 한 줄
   - `001`~`003`을 Firefly 태그 기준으로 rebase

3. **bsp: spacemit-k3: use SoC thermal zone for CPU temp**
   - `packages/bsp/common/usr/lib/armbian/armbian-hardware-monitor`

4. **(follow-up) CTF2301 hwmon scale**
   - 드라이버 패치

### 12.10 남은 주의사항

- 현재 부팅은 **Armbian distro boot**입니다.
  - SPI NOR U-Boot env가 `autoboot`/`bootfunc`로 `/boot.scr`을 찾습니다.
  - `boot.scr`이 `armbianEnv.txt`를 읽고 `Image`/DTB를 로드합니다.
  - `env_k3.txt`는 공장/vendor env 복구용으로만 남겨 두었습니다. Armbian 경로에서는 사용하지 않습니다.
- `CTF2301` hwmon 온도는 10배 스케일로 보입니다.
  - CPU 온도 표시는 `thermal_zone0`을 사용하므로 문제없습니다.
  - hwmon 자체를 고치려면 CTF2301 드라이버 패치가 필요합니다.
- 보드는 여전히 `.wip`입니다.
  - 정식 지원으로 올리려면 `.conf` 승격과 `BOARD_MAINTAINER` 추가가 필요합니다.
- 첫 번째 `k3-br-v1.0.8` 시도는 4~11장에 기록되어 있습니다.
  - 최종 해결은 이 12장입니다.

---

## 13. 의사 결정 로그

이 장은 “왜 이렇게 했는지”를 남긴 기록입니다. 같은 문제를 다시 만났을 때 같은 시행착오를 반복하지 않기 위해 작성했습니다.

> contrib 관점에서 필요한 결론만 뽑으면 `AIBOX-K3-GUIDE.md` 3장(소스 선택 근거)과 9장(기여 준비)입니다.

### 13.1 문제 정의

| 질문 | 답 |
|---|---|
| 팬이 왜 최대 속도인가? | CTF2301 팬 컨트롤러가 probe되지 않아서 |
| 왜 probe되지 않는가? | i2c6이 defer됐기 때문 |
| 왜 i2c6이 defer됐는가? | power-controller가 defer됐기 때문 |
| 왜 power-controller가 defer됐는가? | mpxy_mbox가 없기 때문 |
| 왜 mpxy_mbox가 없나? | SBI MPXY extension이 없기 때문 |
| 왜 SBI MPXY가 없나? | Armbian 펌웨어 세트가 공장 펌웨어와 다른 소스/버전이기 때문 |
| 왜 온도가 335°C인가? | CTF2301 hwmon을 CPU 온도로 읽었기 때문 |

### 13.2 선택지 비교

| 선택지 | 내용 | 장점 | 단점 | 결정 |
|---|---|---|---|---|
| A | 공장 SPI NOR로 복구 | 바로 안정적 | Armbian-only가 아님 | 보류 |
| B | Armbian 펌웨어 세트 수정 | 목표 달성 | 빌드/패치 작업 필요 | **선택** |
| C | 공장 FSBL/OpenSBI/ESOS + Armbian U-Boot | 일부만 Armbian | 버전 혼합 위험 | 보류 |

B를 선택한 이유:

- 사용자의 목표가 “Armbian만으로 부팅 체인을 관리하는 것”이었습니다.
- 팬 문제는 하드웨어 고장이 아니라 펌웨어 세트 불일치였습니다.
- 공장 펌웨어의 소스 태그를 알 수 있었기 때문에 재현 가능한 수정이 가능했습니다.

### 13.3 실제 시도와 결과

| 시도 | 결과 | 교훈 |
|---|---|---|
| 공장 OpenSBI만 되돌림 | MPXY 여전히 실패 | OpenSBI 하나만으로는 부족 |
| 공장 OpenSBI + 공장 ESOS로 되돌림 | MPXY 여전히 실패 | OpenSBI/ESOS만으로는 부족 |
| Armbian `k3-br-v1.0.8` FSBL/OpenSBI/ESOS | MPXY 실패, 팬 최대 | 세트 전체가 오래된 소스 |
| Firefly 태그로 U-Boot/OpenSBI/ESOS 전체 교체 | MPXY 정상, 팬 정상 | **펌웨어는 세트로 맞춰야 함** |

중간에 보드가 부팅하지 않았던 이유:

- 공장 OpenSBI/ESOS와 Armbian FSBL/U-Boot를 섞었을 때 부팅이 불안정했습니다.
- 첫 Firefly 세트는 콜드 부팅 후 정상 동작했지만, 콘솔/USB가 재열거되는 동안 너무 빨리 “죽었다”고 판단한 경우가 있었습니다.
- 그래서 이후에는 전원을 완전히 분리했다가 다시 연결하고, 시리얼 콘솔과 SSH를 충분히 기다린 뒤 판단했습니다.

교훈:

- 부트 펌웨어는 **한 세트**로 맞춰야 합니다.
- 한 파티션씩 바꾸는 테스트는 위험합니다.
- 항상 시리얼 콘솔, USB 다운로드 모드, SPI NOR 백업을 준비해야 합니다.

### 13.4 왜 Firefly 태그인가

공장 SDK manifest:

```text
k3/linux6.18_release_v1.0.0a
```

공장 U-Boot:

```text
e471ad131b riscv: dts: update k3-firefly-common from k3_com260_ifx
```

Armbian이 사용하던 태그:

```text
spacemit-com/uboot-2022.10 k3-br-v1.0.8 (3243c5fc)
```

공장 펌웨어는 Firefly의 최신 K3 릴리스이고, Armbian은 더 오래된 SpacemiT 태그였습니다. 그래서 공장 소스 태그로 맞췄습니다.

### 13.5 왜 distro boot로 전환했나

U-Boot 패치 001~003은 Armbian distro boot(`boot.scr`/`extlinux`)를 위한 패치입니다. Firefly 태그에서는 이 패치들이 그대로 적용되지 않았습니다.

처음에는 팬/MPXY 문제를 먼저 해결하기 위해 vendor env(`nor_boot`/`boot_kernel`)와 `env_k3.txt` 경로를 유지했습니다. 팬/온도가 정상화된 뒤, 001~003을 Firefly 태그 기준으로 다시 작성했습니다.

전환 후:

- U-Boot env가 `autoboot`/`bootfunc`로 `/boot.scr`을 찾습니다.
- `boot.scr`이 `armbianEnv.txt`에서 `fdtfile`/`mtdparts`를 읽습니다.
- 커널/DTB 로드가 Armbian 표준 경로로 바뀝니다.
- `env_k3.txt`는 공장/vendor env 복구용으로만 남습니다.

실제 보드에서 `Loading Armbian environment (scsi 0:1)` → `Booting Armbian (scsi 0:1)` 로그를 확인했습니다.

### 13.6 왜 thermal_zone0인가

`CTF2301`은 팬 컨트롤러이면서 자체 온도 센서를 제공합니다. Linux hwmon으로는 `hwmon0`입니다.

하지만 이 값은:

- CPU 온도가 아님
- 스케일이 SoC 온도 센서와 다름
- 팬 제어용 내부 센서에 가까움

CPU 온도는 `thermal_zone0`이 맞습니다. 그래서 `soctemp`를 `thermal_zone0/temp`로 연결했습니다.

### 13.7 남은 결정

| 항목 | 현재 | 다음 |
|---|---|---|
| U-Boot env | Armbian `autoboot`/`bootfunc` | 유지 |
| 부팅 경로 | `/boot.scr` + `armbianEnv.txt` | 유지 |
| ATF RCPU overlap 패치 | family 기본 유지 | AIBOX-K3에는 빈 patch dir 사용 |
| CTF2301 hwmon 스케일 | BSP에서 `thermal_zone0` 사용 | 드라이버 패치로 10x 스케일 수정 |
| 보드 상태 | `.wip` | 안정화 후 `.conf` 승격 |
| 이미지 | 최종 family/ESOS/BSP/distro boot 이미지 | 클린 설치 패키지로 배포 |

---

## 14. Armbian만으로 빌드하고 설치하는 최종 절차

이 장은 “Firefly SDK 없이 Armbian build framework만으로 이미지를 만들고, AIBOX-K3에 설치하는 방법”입니다.

> **중요:** U-Boot/OpenSBI는 Armbian이 Firefly 소스에서 빌드합니다. ESOS는 벤더 바이너리 blob이라 `packages/blobs`에 들어 있습니다. 커널은 Armbian이 소스에서 빌드합니다. 즉 Firefly SDK를 직접 빌드할 필요는 없습니다.

### 14.1 전체 그림

```text
[빌드 호스트]
  Armbian build framework
    -> U-Boot/OpenSBI (Firefly tag)
    -> ESOS blob (packages/blobs)
    -> Linux kernel (spacemit-com/linux-6.18)
    -> Armbian 이미지 (.img)

[AI Box]
  UFS: bootfs + rootfs
  SPI NOR: bootinfo/fsbl/env/esos/opensbi/uboot
```

### 14.2 준비물

- Ubuntu 빌드 호스트 (16코어 이상 권장)
- Docker 또는 Podman
- AIBOX-K3
- USB-C 케이블 2개
  - Console 포트용
  - 플래싱 포트용
- 시리얼 콘솔 프로그램 (`picocom`)
- TitanTools, `fastboot`, 또는 `armbian-install` (19장)
- 공장 펌웨어/백업 (복구용)

### 14.3 최종 Armbian 트리 준비

변경 파일:

```text
config/boards/firefly-aibox-k3.wip
config/sources/families/spacemit-k3.conf          # ESOS 선택 한 줄만
packages/blobs/riscv64/spacemit-k3/aibox-k3/esos.itb
packages/bsp/common/usr/lib/armbian/armbian-hardware-monitor
patch/u-boot/legacy/u-boot-spacemit-k3-firefly/
patch/atf/atf-spacemit-k3-firefly/
```

`config/boards/firefly-aibox-k3.wip`의 핵심:

```bash
function post_family_config__firefly_aibox_k3_use_firefly_firmware() {
	declare -g BOOTTAG="k3/linux6.18_release_v1.0.0a"
	declare -g ATFSOURCE="https://github.com/Firefly-spacemit-linux/opensbi.git"
	declare -g ATFBRANCH="tag:${BOOTTAG}"
	declare -g ATFPATCHDIR="atf-spacemit-k3-firefly"
	declare -g BOOTSOURCE="https://github.com/Firefly-spacemit-linux/uboot-2022.10.git"
	declare -g BOOTBRANCH="tag:${BOOTTAG}"
	declare -g BOOTPATCHDIR="legacy/u-boot-spacemit-k3-firefly"
	declare -g K3_ESOS_BLOB="${SRC}/packages/blobs/riscv64/spacemit-k3/aibox-k3/esos.itb"
}
```

### 14.4 빌드 호스트 요구사항 설치

처음 한 번만:

```bash
cd ~/code/armbian/build
sudo ./compile.sh requirements
```

### 14.5 전체 이미지 빌드

```bash
./compile.sh build BOARD=firefly-aibox-k3 BRANCH=legacy \
  BUILD_DESKTOP=no BUILD_MINIMAL=no EXPERT=yes KERNEL_CONFIGURE=no RELEASE=resolute
```

이 명령이 하는 일:

1. Firefly 태그에서 U-Boot/OpenSBI 소스를 받아 빌드
2. `packages/blobs/.../esos.itb`를 U-Boot 패키지에 포함
3. `spacemit-com/linux-6.18`에서 커널 빌드
4. BSP 패키지 빌드 (`armbian-hardware-monitor` 온도 수정 포함)
5. bootfs/rootfs와 전체 이미지 생성

캐시가 있으면 약 4~5분, 처음에는 더 오래 걸립니다.

### 14.6 결과 파일

```text
output/images/Armbian-unofficial_..._Firefly-aibox-k3_resolute_legacy_6.18.3.img
output/images/...img.sha
output/images/...img.txt

output/debs/linux-u-boot-firefly-aibox-k3-legacy_..._2022.10-Se471-...deb
```

최종 U-Boot deb:

```text
linux-u-boot-firefly-aibox-k3-legacy_26.11.0-trunk_riscv64__2022.10-Se471-P78cf-Hde31-V0a62-B56a7-R448a.deb
```

### 14.7 이미지 검증

이미지는 GPT + 4096바이트 섹터를 사용합니다.

```bash
cd output/images
IMG=$(ls -t *.img | head -1)
sudo losetup -b 4096 -fP --show "$IMG"
```

예:

```text
/dev/loop22
```

bootfs:

```bash
sudo mount -o ro /dev/loop22p1 /mnt/aibox-boot
ls -l /mnt/aibox-boot/boot.scr /mnt/aibox-boot/armbianEnv.txt
cat /mnt/aibox-boot/armbianEnv.txt
```

정상:

```text
boot.scr
armbianEnv.txt
fdtfile=spacemit/k3-firefly-aibox-k3.dtb
extraargs=... mtdparts=...
```

rootfs:

```bash
sudo mount -o ro /dev/loop22p2 /mnt/aibox-root
ls -l /mnt/aibox-root/usr/sbin/flashcp
grep -n write_uboot_platform_mtd /mnt/aibox-root/usr/lib/u-boot/platform_install.sh
grep -n 'spacemit-k3' /mnt/aibox-root/usr/lib/armbian/armbian-hardware-monitor
chroot /mnt/aibox-root dpkg -l | grep -E 'mtd-utils|linux-u-boot'
```

정리:

```bash
sudo umount /mnt/aibox-boot /mnt/aibox-root
sudo losetup -d /dev/loop22
```

### 14.8 bootfs/rootfs 추출

TitanTools 파티션 방식에 쓰려면 이미지에서 두 파티션을 추출합니다.

```bash
sudo losetup -b 4096 -fP --show "$IMG"
sudo dd if=/dev/loop22p1 of=bootfs.img bs=4M status=progress
sudo dd if=/dev/loop22p2 of=rootfs.ext4 bs=4M status=progress
sudo losetup -d /dev/loop22
```

### 14.9 UFS에 쓰기 (TitanTools)

1. AI Box 전원 분리
2. 플래싱 USB-C 포트를 호스트에 연결
3. Boot 버튼을 누른 상태에서 전원 연결
4. TitanTools 실행
5. `Dev Tools → USB Download → Scan Devices`
6. `partition_universal.json` 선택
7. `bootfs.img`, `rootfs.ext4`를 새 파일로 교체
8. `Start Flashing`

> UFS의 hidden U-Boot 파티션은 SPI NOR 부팅에는 사용되지 않습니다. bootfs/rootfs가 핵심입니다.

Armbian 표준 도구로 설치하려면 TitanTools 대신 **19장의 `armbian-install`**을 사용하세요. `armbian-install`은 live USB에서 UFS와 SPI NOR을 함께 씁니다.

### 14.10 공장 SPI NOR로 첫 부팅

SPI NOR은 아직 공장 펌웨어입니다. 이 상태에서 새 이미지가 부팅되는지 먼저 확인합니다.

```bash
ssh leopark@192.168.1.70
ls -l /boot/boot.scr /boot/armbianEnv.txt /boot/env_k3.txt
cat /proc/mtd
dpkg -l | grep -E 'mtd-utils|linux-u-boot'
```

정상:

```text
boot.scr 있음
armbianEnv.txt에 fdtfile + mtdparts
/proc/mtd 6개 파티션
mtd-utils installed
linux-u-boot ...P78cf-Hde31-V0a62... installed
```

`boot.scr`이 없으면 SPI NOR을 쓰지 마세요.

### 14.11 SPI NOR에 Armbian 펌웨어 쓰기

#### 방법 A: Linux에서 함수 사용 (권장)

새 이미지에는 새 U-Boot deb가 들어 있습니다.

```bash
sudo bash -c '
  set -e
  source /usr/lib/u-boot/platform_install.sh
  write_uboot_platform_mtd "$DIR" /dev/mtd0 /tmp/spinor.log
'
cat /tmp/spinor.log
```

정상:

```text
SPI NOR bootloader written
```

#### 방법 B: 수동 flashcp

```bash
cd /usr/lib/linux-u-boot-legacy-firefly-aibox-k3

sudo flashcp -v bootinfo_spinor.bin     /dev/mtd1
sudo flashcp -v FSBL.bin                 /dev/mtd2
sudo flashcp -v u-boot-env-default.bin   /dev/mtd3
sudo flashcp -v esos.itb                 /dev/mtd4
sudo flashcp -v fw_dynamic.itb           /dev/mtd5
sudo flashcp -v u-boot.itb               /dev/mtd6
```

#### 방법 C: TitanTools + partition_4M.json

1. AI Box 전원 분리
2. 플래싱 USB-C 포트 연결
3. Boot 버튼 누른 채 전원 연결
4. TitanTools에서 `partition_4M.json` 선택
5. 같은 디렉터리에 `bootinfo_spinor.bin`, `FSBL.bin`, `u-boot-env-default.bin`, `esos.itb`, `fw_dynamic.itb`, `u-boot.itb` 배치
6. `Start Flashing`

#### 방법 D: fastboot 직접 사용

```bash
fastboot devices -l
fastboot stage FSBL.bin
fastboot continue
sleep 10
fastboot oem speed:super-speed
fastboot stage u-boot.itb
fastboot continue
sleep 3
fastboot flash mtd partition_4M.json
fastboot reboot
```

### 14.12 재부팅 후 확인

```bash
dmesg | grep -iE 'mpxy|ctf|thermal'
cat /sys/class/thermal/thermal_zone0/temp
cat /sys/class/hwmon/hwmon0/fan1_input
cat /sys/class/hwmon/hwmon0/pwm1
```

정상:

```text
mailbox registered with 5 channels
thermal_zone0=35000
fan1_input=548
pwm1=16
```

welcome 메시지:

```text
CPU temp: 35°C
```

### 14.13 복구

1. AI Box 전원 분리
2. 플래싱 USB-C 포트 연결
3. Boot 버튼 누른 채 전원 연결
4. TitanTools로 공장 펌웨어 플래싱
5. 또는 `spi-flash-howto.txt`의 fastboot 순서

백업:

`write_uboot_platform_mtd()`는 자동으로 백업하지 않습니다. SPI NOR를 쓰기 전에 직접 백업하세요.

```bash
sudo bash <<'EOF'
out=/var/backups/spinor-$(date +%Y%m%d-%H%M%S)
mkdir -p "$out"
for name in bootinfo fsbl env esos opensbi uboot; do
  dev=$(awk -v n="\"$name\"" '$NF == n { sub(":", "", $1); print "/dev/" $1 }' /proc/mtd)
  dd if="$dev" of="$out/$name.bin" bs=64k status=none
done
tar -C /var/backups -czf "$out.tgz" "$(basename "$out")"
echo "$out.tgz"
EOF
```

호스트로 복사:

```bash
scp leopark@192.168.1.70:/var/backups/spinor-*.tgz .
```

### 14.14 최종 체크리스트

- [ ] 설치 방법 선택: `armbian-install`(19장) 또는 TitanTools(17장)
- [ ] target 장치가 UFS인지 `lsblk`로 확인
- [ ] 빌드 호스트에서 `./compile.sh requirements`
- [ ] `./compile.sh build ...` 성공
- [ ] 이미지 `boot.scr`, `armbianEnv.txt`, `mtdparts` 확인
- [ ] rootfs에 `mtd-utils`, 새 U-Boot deb, 온도 수정 확인
- [ ] TitanTools로 UFS에 bootfs/rootfs 쓰기
- [ ] 공장 SPI NOR로 첫 부팅 확인
- [ ] SPI NOR 백업을 호스트로 복사
- [ ] `write_uboot_platform_mtd`로 SPI NOR 쓰기
- [ ] 재부팅 후 `boot.scr`/`armbianEnv.txt` 부팅 확인
- [ ] 재부팅 후 MPXY/팬/온도 확인
- [ ] `systemctl --failed` 0
- [ ] 복구 경로 확인

---

## 15. 변경 파일과 diff

### 15.1 보드 설정

`config/boards/firefly-aibox-k3.wip`:

- `BOOTSCRIPT`, `BOOTENV_FILE`
- `PACKAGE_LIST_BOARD="mtd-utils"`
- `extraargs`에 `mtdparts`
- `write_uboot_platform_mtd()`

전체 내용은 5.6을 참고하세요.

### 15.2 AIBOX-K3 전용 Firefly 설정

`config/sources/families/spacemit-k3.conf`의 기본값은 바꾸지 않습니다.

```text
BOOTTAG="k3-br-v1.0.8"
ATFSOURCE="https://github.com/spacemit-com/opensbi.git"
BOOTSOURCE="https://github.com/spacemit-com/uboot-2022.10.git"
BOOTPATCHDIR="legacy/u-boot-spacemit-k3"
```

AIBOX-K3만 `config/boards/firefly-aibox-k3.wip`의 `post_family_config` hook에서 override합니다.

```bash
BOOTTAG="k3/linux6.18_release_v1.0.0a"
ATFSOURCE="https://github.com/Firefly-spacemit-linux/opensbi.git"
ATFBRANCH="tag:${BOOTTAG}"
ATFPATCHDIR="atf-spacemit-k3-firefly"
BOOTSOURCE="https://github.com/Firefly-spacemit-linux/uboot-2022.10.git"
BOOTBRANCH="tag:${BOOTTAG}"
BOOTPATCHDIR="legacy/u-boot-spacemit-k3-firefly"
```

family에는 ESOS blob 선택 변수 한 줄만 추가합니다.

```bash
local esos_blob="${K3_ESOS_BLOB:-${SRC}/packages/blobs/riscv64/spacemit-k3/esos.itb}"
```

### 15.3 ESOS blob (AIBOX-K3 전용)

```text
packages/blobs/riscv64/spacemit-k3/esos.itb            450,419 (family 기본)
packages/blobs/riscv64/spacemit-k3/aibox-k3/esos.itb   506,499 (AIBOX-K3)
```

출처:

```text
/home/leopark/code/k3-buildroot-sdk/output/firefly-k3/images/esos.itb
```

### 15.4 U-Boot 패치 디렉터리

새 디렉터리:

```text
patch/u-boot/legacy/u-boot-spacemit-k3-firefly/
  001-SpacemiT-K3-fixup-boot-variables.patch
  002-Update-SpacemiT-K3-defconfig.patch
  003-Update-SpacemiT-K3-Env.patch
  004-board-spacemit-k3-Initialize-miiphy-early-to-fix-EFI.patch
  005-lib-printf-Revert-spacemit-s-buggy-CONFIG_PRINT_TIME.patch
  006-Fixup-circular-deps.patch
```

`001`~`003`은 Firefly 태그 기준으로 rebase했습니다. 세 패치는 `git apply`로 clean apply되고, 빌드된 U-Boot env는 `bootcmd=run autoboot;`/`bootfunc`를 사용합니다.

### 15.5 ATF 패치

`patch/atf/atf-spacemit-k3/001-k3-fix-rcpu1-reserved-mem-overlap.patch`는 family 기본으로 유지합니다.

AIBOX-K3는 Firefly OpenSBI를 쓰므로 이 패치를 적용하지 않습니다. 대신 `patch/atf/atf-spacemit-k3-firefly/`(`.gitkeep`만 있는 빈 패치 디렉터리)를 `ATFPATCHDIR`로 지정합니다. 다른 K3 보드는 기존 ATF 패치를 그대로 사용합니다.

> 주의: 이 디렉터리에 `README.md` 같은 파일을 추가하면 U-Boot deb 버전 문자열이 바뀝니다
> (패치 디렉터리 해시가 버전 문자열에 포함). 산출물까지 다시 만들어야 하므로, 정리 작업은
> PR 직전에 한 번에 하고 전체를 재빌드하세요. (`AIBOX-K3-GUIDE.md` 9.2 참고)

### 15.6 BSP 온도 수정

`packages/bsp/common/usr/lib/armbian/armbian-hardware-monitor`:

```diff
-	# SoC temp: check sun7i legacy exception first, then standard path(s)
-	if [[ -d "/sys/devices/platform/a20-tp-hwmon" ]]; then
+	# SoC temp: check board-specific exceptions first, then standard path(s)
+	if [[ ${BOARD} == "firefly-aibox-k3" ]]; then
+		# CTF2301 (hwmon0) is the fan controller, not the SoC sensor.
+		ln -fs /sys/devices/virtual/thermal/thermal_zone0/temp /etc/armbianmonitor/datasources/soctemp
+	elif [[ -d "/sys/devices/platform/a20-tp-hwmon" ]]; then
```

동작 설명:

1. `armbian-hardware-monitor`의 `prepare_temp_monitoring()`이 부팅 시 실행됩니다.
2. 이 함수는 `/etc/armbianmonitor/datasources/` 아래에 심링크를 만듭니다
   (`soctemp`, `pmictemp`).
3. `armbianmonitor`가 그 심링크를 읽어 MOTD에 "CPU temp: 39°C"처럼 표시합니다.

기본 규칙은 `hwmon0/temp1_input`입니다. AIBOX-K3에서 `hwmon0`은 CTF2301 팬 컨트롤러라서

```text
/sys/class/hwmon/hwmon0/temp1_input = 345625   # 10배 스케일, SoC 온도 아님
/sys/class/thermal/thermal_zone0/temp = 35000   # 실제 SoC 온도(35°C)
```

가 되어, 기본 규칙을 쓰면 345°C가 표시됩니다. 그래서 AIBOX-K3만 `thermal_zone0`을 씁니다.

검증:

```bash
readlink -f /etc/armbianmonitor/datasources/soctemp
cat /etc/armbianmonitor/datasources/soctemp
```

자세한 설명은 `AIBOX-K3-GUIDE.md` 5장을 보세요.

### 15.7 변경하지 않은 것

- 커널 소스/설정은 그대로
- UFS 파티션 레이아웃은 그대로
- `partition_4M.json`/`partition_universal.json` 레이아웃은 그대로
- 보드 이름/DTB는 그대로

---

## 16. 최종 커밋과 PR

### 16.1 커밋 계획

1. **board: firefly-aibox-k3: add SPI NOR bootloader update**
   - `config/boards/firefly-aibox-k3.wip`

2. **board: firefly-aibox-k3: use Firefly firmware and distro boot**
   - `config/boards/firefly-aibox-k3.wip`의 `post_family_config` hook
   - `packages/blobs/riscv64/spacemit-k3/aibox-k3/esos.itb`
   - `patch/u-boot/legacy/u-boot-spacemit-k3-firefly/` (`001`~`006`)
   - `patch/atf/atf-spacemit-k3-firefly/`
   - `config/sources/families/spacemit-k3.conf`의 ESOS 선택 한 줄

3. **bsp: spacemit-k3: use SoC thermal zone for CPU temp**
   - `packages/bsp/common/usr/lib/armbian/armbian-hardware-monitor`

4. **(follow-up) CTF2301 hwmon scale**
   - 드라이버 패치

### 16.2 PR 제목

```text
board: firefly-aibox-k3: add SPI NOR and distro boot support
```

### 16.3 PR 본문

10.3의 “최종 PR 본문”을 사용하세요.

### 16.4 PR 체크리스트

- [x] 보드 설정 변경
- [x] U-Boot/OpenSBI Firefly 태그 빌드
- [x] 보드 전용 ESOS blob (506,499)
- [x] 다른 K3 보드 기본값 유지 확인
- [x] SPI NOR 쓰기/검증
- [x] MPXY 정상
- [x] 팬 제어 정상
- [x] CPU 온도 표시 정상
- [x] `systemctl --failed` 0
- [x] U-Boot 001~003 rebase
- [x] `boot.scr`/`armbianEnv.txt` distro boot
- [ ] ATF 패치 rebase/제거 (follow-up)
- [ ] `.wip` → `.conf` 승격 (follow-up)

### 16.5 최종 주의사항

- 현재 부팅은 Armbian distro boot입니다. U-Boot env가 `/boot.scr`을 찾습니다.
- `env_k3.txt`는 공장/vendor env 복구용으로만 남아 있습니다. Armbian 경로에서는 사용하지 않습니다.
- 이미지는 최종 family/ESOS/BSP/distro boot 변경으로 **재빌드**했습니다.
- 보드는 여전히 `.wip`입니다.
- 팬/온도 문제는 해결됐지만, CTF2301 hwmon 자체의 10x 스케일은 별도 드라이버 이슈입니다.

---

## 17. 클린 설치용 TitanTools 펌웨어 패키지

이 장은 “Armbian만으로 빌드한 이미지 + Armbian이 빌드한 SPI NOR 펌웨어”를 **TitanTools가 읽을 수 있는 하나의 펌웨어 패키지**로 만드는 방법입니다.

참고한 공장 패키지:

```text
/var/home/leopark/Downloads/Armbian-firefly-aibox-k3-titan/
```

이 디렉터리는 Firefly 공장 펌웨어를 TitanTools로 플래싱하기 위한 레이아웃입니다. 우리는 **JSON/디렉터리 구조는 그대로** 두고, 안의 바이너리만 Armbian 빌드 결과로 바꿉니다.

### 17.1 공장 패키지 구조 읽기

```text
Armbian-firefly-aibox-k3-titan/
├── bootfs.img                    256MiB FAT boot 파티션
├── rootfs.ext4                   루트 파일시스템
├── env.bin                       U-Boot env (16KiB)
├── esos.itb                      ESOS 펌웨어
├── fw_dynamic.itb                OpenSBI
├── u-boot.itb                    U-Boot proper
├── factory/
│   ├── bootinfo_block.bin        UFS/GPT용 bootinfo
│   ├── bootinfo_spinand.bin      SPI NAND용 bootinfo
│   ├── bootinfo_spinor.bin       SPI NOR용 bootinfo
│   └── FSBL.bin                  U-Boot SPL
├── partition_4M.json             SPI NOR 레이아웃
├── partition_64M.json            SPI NAND 레이아웃 (참고)
├── partition_flash.json          GPT/raw 플래시 레이아웃 (참고)
├── partition_universal.json      UFS GPT 레이아웃
├── fastboot.yaml                 TitanTools multi-flash 설명
└── genimage.cfg                  디스크 이미지 생성용 (참고)
```

각 JSON의 역할:

| 파일 | 용도 |
|---|---|
| `partition_universal.json` | UFS에 GPT + hidden firmware + bootfs/rootfs를 씀 |
| `partition_4M.json` | SPI NOR 8MiB에 bootinfo/fsbl/env/esos/opensbi/uboot를 씀 |
| `partition_64M.json` | SPI NAND 64MiB용 (AIBOX-K3 NOR에는 사용하지 않음) |
| `partition_flash.json` | GPT/raw 전체 플래시용 |
| `fastboot.yaml` | `mtd-size`/`blk-size`에 따라 `partition_4M.json`/`partition_64M.json`을 선택하는 TitanTools 설정 |
| `genimage.cfg` | `genimage`로 디스크 이미지를 만들 때 사용 |

중요한 점:

- `partition_universal.json`의 hidden `uboot` 파티션은 **UFS 안의 파티션**입니다.
- AIBOX-K3는 **SPI NOR에서 부팅**하므로, 실제 부트로더는 `partition_4M.json`이 쓰는 SPI NOR 쪽입니다.
- UFS hidden firmware는 NOR 부팅 경로에서 직접 사용되지 않지만, 레이아웃을 맞추기 위해 함께 씁니다.

### 17.2 Armbian만으로 최종 이미지 빌드

빌드 호스트:

```bash
cd ~/code/armbian/build
./compile.sh build BOARD=firefly-aibox-k3 BRANCH=legacy \
  BUILD_DESKTOP=no BUILD_MINIMAL=no EXPERT=yes KERNEL_CONFIGURE=no RELEASE=resolute
```

이번 최종 이미지:

```text
output/images/Armbian-unofficial_26.11.0-trunk_Firefly-aibox-k3_resolute_legacy_6.18.3.img
SHA256: 6def878d1a589e451a360e333c1e08dbf12ecc8f045f8ba58d6eb9f271991529
```

이미지 안에 포함된 것:

```text
linux-u-boot-firefly-aibox-k3-legacy = 2022.10-Se471-P78cf-Hde31-V0a62-...
mtd-utils
write_uboot_platform_mtd()
armbian-hardware-monitor spacemit-k3 온도 수정
boot.scr + armbianEnv.txt (distro boot)
esos.itb = 506,499 bytes
```

즉 이 이미지는 **최종 Firefly 태그 펌웨어 + distro boot + 팬/온도 수정**을 포함합니다.

### 17.3 최종 패키지 만들기

빌드 호스트에서:

```bash
cd ~/code/armbian/build

IMG=$(ls -t output/images/*.img | head -1)
DEB=$(ls -t output/debs/linux-u-boot-firefly-aibox-k3-legacy_*.deb | head -1)
D=$HOME/armbian-firefly-aibox-k3-clean

rm -rf "$D"
mkdir -p "$D/factory"

# 1) 이미지에서 bootfs/rootfs 추출
LOOP=$(sudo losetup -b 4096 -fP --show "$IMG")
sudo dd if=${LOOP}p1 of="$D/bootfs.img" bs=4M status=progress
sudo dd if=${LOOP}p2 of="$D/rootfs.ext4" bs=4M status=progress
sudo losetup -d "$LOOP"

# 2) U-Boot deb에서 SPI NOR 파일 추출
rm -rf /tmp/final-uboot
mkdir -p /tmp/final-uboot
dpkg-deb -x "$DEB" /tmp/final-uboot
U=/tmp/final-uboot/usr/lib/linux-u-boot-legacy-firefly-aibox-k3

cp "$U/u-boot-env-default.bin" "$D/env.bin"
cp "$U/esos.itb"               "$D/"
cp "$U/fw_dynamic.itb"         "$D/"
cp "$U/u-boot.itb"             "$D/"
cp "$U/FSBL.bin"               "$D/"
cp "$U/bootinfo_spinor.bin"    "$D/"

cp "$U/FSBL.bin"               "$D/factory/FSBL.bin"
cp "$U/bootinfo_block.bin"     "$D/factory/"
cp "$U/bootinfo_spinor.bin"    "$D/factory/"
cp "$U/bootinfo_spinand.bin"   "$D/factory/"

# 3) 공장 패키지의 JSON/설정은 로컬에서 빌드 호스트로 복사
# 로컬 호스트:
tar -C /var/home/leopark/Downloads/Armbian-firefly-aibox-k3-titan -czf /tmp/ref-json.tar.gz \
  partition_4M.json partition_64M.json partition_flash.json partition_universal.json \
  fastboot.yaml genimage.cfg
scp /tmp/ref-json.tar.gz leopark@192.168.1.10:/tmp/ref-json.tar.gz

# 빌드 호스트:
tar -xzf /tmp/ref-json.tar.gz -C "$D"
```

최종 디렉터리:

```text
armbian-firefly-aibox-k3-clean/
├── bootfs.img                    268,435,456
├── rootfs.ext4                   2,649,751,552
├── env.bin                       16,384
├── esos.itb                      506,499
├── fw_dynamic.itb                271,311
├── u-boot.itb                    2,683,538
├── FSBL.bin                      453,984
├── bootinfo_spinor.bin           80
├── factory/
│   ├── bootinfo_block.bin        80
│   ├── bootinfo_spinand.bin      80
│   ├── bootinfo_spinor.bin       80
│   └── FSBL.bin                  453,984
├── partition_4M.json
├── partition_64M.json
├── partition_flash.json
├── partition_universal.json
├── fastboot.yaml
└── genimage.cfg
```

파일 대응:

| 패키지 파일 | 출처 |
|---|---|
| `bootfs.img` | Armbian 이미지 p1 |
| `rootfs.ext4` | Armbian 이미지 p2 |
| `env.bin` | `u-boot-env-default.bin` |
| `esos.itb` | U-Boot deb의 ESOS blob (506,499) |
| `fw_dynamic.itb` | U-Boot deb의 OpenSBI |
| `u-boot.itb` | U-Boot deb의 U-Boot proper |
| `FSBL.bin` | U-Boot deb의 SPL |
| `bootinfo_spinor.bin` | U-Boot deb의 SPI NOR bootinfo |
| `factory/FSBL.bin` | `FSBL.bin` 복사 |
| `factory/bootinfo_*.bin` | U-Boot deb의 bootinfo 변형 |

### 17.4 압축 파일 만들기

```bash
cd $HOME
zip -r -q -1 armbian-firefly-aibox-k3-clean.zip armbian-firefly-aibox-k3-clean
sha256sum armbian-firefly-aibox-k3-clean.zip > armbian-firefly-aibox-k3-clean.zip.sha
```

이번 결과:

```text
armbian-firefly-aibox-k3-clean.zip
size: 929M
sha256: f20a0aa3aeeb7dcbd49b77c1703af1297ba2448f9aa716bb8c3755b65d439a1d
```

로컬 복사:

```text
/var/home/leopark/Downloads/armbian-firefly-aibox-k3-clean/
  armbian-firefly-aibox-k3-clean.zip
  armbian-firefly-aibox-k3-clean.zip.sha
```

### 17.5 TitanTools로 클린 설치

#### 준비

- AI Box 전원 분리
- **Console 포트**를 호스트에 연결 (시리얼 로그)
- **플래싱 포트(USB 3.0 Type-C DRD)**를 호스트에 연결
- TitanTools 실행

#### 1) UFS 쓰기

1. AI Box 전원 분리
2. 플래싱 포트 연결
3. Boot 버튼을 누른 상태에서 전원 연결
4. 2~3초 후 Boot 버튼에서 손을 뗌
5. TitanTools: `Dev Tools → USB Download → Scan Devices`
6. `armbian-firefly-aibox-k3-clean.zip` 선택
7. `Configure Partition File`에서 `partition_universal.json` 선택
8. `Start Flashing`
9. 완료 후 재부팅

이 단계에서는 SPI NOR은 아직 공장 펌웨어입니다. 공장 U-Boot는 `env_k3.txt`로 부팅합니다. 이미지에 `boot.scr`이 있는지 먼저 확인하세요. 두 번째 단계에서 SPI NOR을 쓰면 Armbian U-Boot가 `boot.scr`로 부팅합니다.

#### 2) SPI NOR 쓰기

1. 다시 전원 분리
2. 플래싱 포트 연결
3. Boot 버튼 누른 채 전원 연결
4. TitanTools에서 같은 패키지 선택
5. `Configure Partition File`에서 `partition_4M.json` 선택
6. `Start Flashing`
7. 재부팅

이제 SPI NOR에 Armbian이 빌드한 Firefly 태그 펌웨어가 들어갑니다.

> `partition_universal.json`과 `partition_4M.json`을 한 번에 플래싱하는 `fastboot.yaml` 흐름도 있습니다. 하지만 초보자에게는 **UFS 먼저, SPI NOR 나중에**가 안전합니다.

### 17.6 fastboot로 설치

TitanTools 대신 fastboot를 쓸 수 있습니다.

UFS:

```bash
fastboot devices -l
fastboot stage factory/FSBL.bin
fastboot continue
sleep 10
fastboot oem speed:super-speed
fastboot stage u-boot.itb
fastboot continue
sleep 3
fastboot flash mtd partition_universal.json
fastboot reboot
```

SPI NOR:

```bash
fastboot devices -l
fastboot stage FSBL.bin
fastboot continue
sleep 10
fastboot oem speed:super-speed
fastboot stage u-boot.itb
fastboot continue
sleep 3
fastboot flash mtd partition_4M.json
fastboot reboot
```

### 17.7 설치 후 확인

시리얼 콘솔:

```text
Loading K3 environment
Loading Armbian environment (scsi 0:1)
Booting Armbian (scsi 0:1)
Starting kernel ...
...
mailbox registered with 5 channels
```

SSH:

```bash
cat /sys/class/thermal/thermal_zone0/temp
cat /sys/class/hwmon/hwmon0/fan1_input
cat /sys/class/hwmon/hwmon0/pwm1
dmesg | grep -iE 'mpxy|ctf'
systemctl --failed
```

정상:

```text
thermal_zone0=35000
fan1_input=548
pwm1=16
systemctl --failed: 0
```

MOTD:

```text
CPU temp: 35°C
```

### 17.8 복구

클린 설치 중 실패하면:

1. AI Box 전원 분리
2. 플래싱 포트 연결
3. Boot 버튼 누른 채 전원 연결
4. TitanTools로 공장 펌웨어 플래싱
5. 또는 `spi-flash-howto.txt`의 fastboot 순서

SPI NOR 백업은 `write_uboot_platform_mtd()`가 자동으로 만들지 않습니다. 쓰기 전에 `14.13`의 수동 백업 명령을 실행하세요. 백업 파일 예:

```text
/var/backups/spinor-<시각>.tgz
```

### 17.9 커밋/PR은 어떻게 할까

이번 요청에서는 **커밋/푸시/PR을 실제로 진행하지 않았습니다.** 문서에만 정리합니다.

### 17.9.1 Armbian upstream에 올리는 정식 경로

`armbian/build`는 **GitLab 내부 미러를 GitHub에 자동 반영하는 mirror**입니다.
그래서 Armbian 멤버라고 해서 직접 push가 되는 곳이 아닙니다.

```text
ERROR: Permission to armbian/build.git denied to leoppark94.
```

→ **fork에 push하고, PR을 여는 방식**입니다.

```bash
# 1) fork를 remote으로 등록 (이미 있으면 set-url)
git remote add fork git@github.com:leoppark94/armbian_build.git
git remote -v

# 2) DCO sign-off 를 붙여 커밋 (Armbian은 GPG가 아니라 sign-off를 요구)
git commit --amend -s --no-edit

# 3) fork에 push
git push -u fork feature/firefly-aibox-k3

# 4) PR
#    https://github.com/armbian/build/compare/main...leoppark94:armbian_build:feature/firefly-aibox-k3
```

주의:

- Armbian은 **DCO sign-off(`Signed-off-by:` trailer)** 를 요구합니다. GPG/SSH 서명은 요구하지 않습니다.
  `git commit -s` 또는 `git commit --amend -s`로 trailer를 붙이세요.
- `--amend`하면 커밋 SHA가 바뀝니다. 이미 push한 브랜치라면 `--force-with-lease`가 필요합니다.
- fork의 `main`이 upstream보다 뒤처져 있을 수 있습니다. PR 전에 맞춰 주세요.

  ```bash
  git fetch origin
  git push fork origin/main:main
  ```

### 17.9.2 서명 설정에 대해

```text
gpg.format=ssh
user.signingkey=/var/home/leopark/.ssh/id_ed25519.pub
commit.gpgsign=true
```

이 설정은 **정상 동작**합니다. git 2.34+ 는 `.pub` 경로를 받아서 같은 이름의 개인키를 찾습니다.
다만 로컬에서 서명을 *검증*하려면 allowed signers 파일이 필요합니다.

```bash
git config --global gpg.ssh.allowedSignersFile ~/.ssh/allowed_signers
printf 'joohyung@proton.me %s\n' "$(cat ~/.ssh/id_ed25519.pub)" > ~/.ssh/allowed_signers
git verify-commit HEAD
```

### 17.9.3 로컬 ref가 꼬였을 때

btrfs 홈 디렉터리에서 브랜치 ref가 손상되면 아래 증상이 납니다.

```text
error: src refspec <branch> does not match any
git status  -> "No commits yet on <branch>"
```

이 경우 ref 이름이 다른 새 브랜치로 옮기면 복구됩니다.

```bash
git pack-refs --all          # 느슨한 ref를 packed-refs로
git checkout -B feature/firefly-aibox-k3 4646b10de
git branch -D <깨진 브랜치>
git status                   # "No commits yet" 가 사라졌는지 확인
```

PR 전에 확인할 것:

- [ ] `config/boards/firefly-aibox-k3.wip`의 `post_family_config` hook
- [ ] `packages/blobs/.../aibox-k3/esos.itb` 506,499
- [ ] `patch/u-boot/legacy/u-boot-spacemit-k3-firefly/`
- [ ] `patch/atf/atf-spacemit-k3-firefly/` (빈 디렉터리)
- [ ] family ESOS 선택 한 줄
- [ ] `armbian-hardware-monitor`의 `${BOARD}` 조건
- [ ] `./compile.sh build ...` 성공
- [ ] `k3picoitx` U-Boot 빌드가 기존 `spacemit-com`/old patch dir을 쓰는지 확인
- [ ] SPI NOR 6개 파티션 MATCH
- [ ] `boot.scr`/`armbianEnv.txt` distro boot 확인
- [ ] MPXY/팬/온도 정상
- [ ] `systemctl --failed` 0
- [ ] DCO sign-off trailer 포함

### 17.10 현재 산출물

```text
[로컬]
/var/home/leopark/Downloads/armbian-firefly-aibox-k3-clean/
  armbian-firefly-aibox-k3-clean.zip
  armbian-firefly-aibox-k3-clean.zip.sha

[로컬 문서]
/var/home/leopark/Downloads/aibox-k3-spi-nor-20261019/
  ARMBIAN-AIBOX-K3-SPI-NOR.ko.md
  uboot-firefly/linux-u-boot-...P78cf-Hde31-V0a62...deb

[빌드 호스트]
/home/leopark/armbian-firefly-aibox-k3-clean/
  (패키지 디렉터리)
/home/leopark/armbian-firefly-aibox-k3-clean.zip
```

---

## 18. 최종 부팅 경로와 선택 기록

### 18.1 최종 부팅 경로

지금 보드가 실제로 쓰는 경로:

```text
SPI NOR U-Boot
  -> autoboot
    -> bootfunc
      -> /boot.scr (scsi 0:1)
        -> armbianEnv.txt
          -> Image + DTB
            -> Linux
```

`/boot.scr`은 이미지의 `BOOTSCRIPT`/`BOOTENV_FILE` 설정으로 생성됩니다. `armbianEnv.txt`에는 `fdtfile`과 `mtdparts`가 들어 있습니다.

### 18.2 2번을 선택한 이유

2번은 U-Boot 패치 `001`~`003`을 Firefly 태그 기준으로 rebase하는 작업입니다.

- `001`: `boot_device`/`boot_devnum`/`bootfs_part`를 `devtype`/`devnum`/`distro_bootpart`로 변경
- `002`: distro boot defconfig 옵션 추가
- `003`: vendor env를 Armbian `autoboot`/`bootfunc` env로 교체

이 작업으로 `BOOTSCRIPT`/`BOOTENV_FILE`/`mtdparts` hook이 실제 부팅에 사용됩니다. Armbian 표준 경로이고, extlinux/EFI 확장도 가능합니다.

### 18.3 1번은 왜 하지 않았나

1번은 vendor env 전용으로 코드를 최소화하는 선택입니다. `BOOTSCRIPT`, `BOOTENV_FILE`, `K3_MTDPARTS`, `pre_install_kernel_debs__extra_boot_args`를 제거하고 `env_k3.txt`만 남깁니다.

2번을 완료했으므로 1번은 필요 없습니다. `env_k3.txt`는 공장/vendor env 복구용으로만 남겨 두었습니다.

### 18.4 검증 결과

실제 보드에서 확인한 것:

```text
U-Boot SPL 2022.10_armbian-2022.10-Se471-P78cf-Hde31-V0a62-B56a7-R448a
U-Boot 2022.10_armbian-2022.10-Se471-P78cf-Hde31-V0a62-B56a7-R448a
Loading K3 environment
Loading Armbian environment (scsi 0:1)
Booting Armbian (scsi 0:1)
Starting kernel ...
```

추가 확인:

- `/proc/cmdline`이 `boot.scr` 경로의 cmdline으로 변경됨
- `mtdparts`가 cmdline에 포함됨
- MPXY: `mailbox registered with 5 channels`
- 팬: `ctf2301_fan`, PWM 16~60/255
- CPU 온도: `thermal_zone0`, 35~39°C
- `systemctl --failed`: 0

### 18.5 남은 후속 작업

- CTF2301 hwmon의 10x 온도 스케일 드라이버 수정
- `.wip` → `.conf` 승격, `BOARD_MAINTAINER` 추가
- upstream PR 제출

---

## 19. Armbian installer로 설치

이 장은 Armbian이 제공하는 `armbian-install`로 UFS와 SPI NOR을 함께 설치하는 방법입니다.

### 19.1 전제

- SPI NOR에 **Armbian U-Boot가 이미 있어야** 합니다.
  - 공장 SPI NOR만 있으면 USB 부팅이 안 될 수 있습니다.
  - 이때는 17장의 TitanTools로 SPI NOR을 먼저 쓰세요.
- Armbian 이미지를 쓴 **live USB**가 필요합니다.
- UFS는 `/dev/sda`로 보입니다.
- SPI NOR 백업을 먼저 뜹니다. `14.13` 참고.

### 19.2 live USB 만들기

빌드한 이미지를 USB에 씁니다.

```bash
IMG=$(ls -t output/images/*.img | head -1)
lsblk -o NAME,SIZE,TYPE,MODEL

sudo dd if="$IMG" of=/dev/sdX bs=4M status=progress conv=fsync
```

`/dev/sdX`는 USB 장치입니다. AIBOX의 UFS(`/dev/sda`)가 아닙니다.

### 19.3 USB로 부팅

AIBOX 전원을 끄고 USB를 연결한 뒤 전원을 켭니다. Armbian U-Boot env는 USB를 먼저 스캔합니다.

시리얼 로그에서 확인:

```text
Loading K3 environment
Loading Armbian environment (usb 0:1)
Booting Armbian (usb 0:1)
Starting kernel ...
```

USB로 부팅했는지 확인:

```bash
findmnt -no SOURCE /
lsblk -o NAME,SIZE,TYPE,MOUNTPOINTS
```

`/`가 USB이고, UFS가 `/dev/sda`로 보여야 합니다.

### 19.4 설치

대화형:

```bash
sudo armbian-install
```

- target: UFS (`/dev/sda`)
- boot: `SPI/MTD`
- filesystem: `ext4`

CLI:

```bash
lsblk -o NAME,SIZE,TYPE,MODEL,MOUNTPOINTS
sudo armbian-install --target /dev/sda --boot mtd --fs ext4 --yes
```

`--boot mtd`는 SPI NOR에 U-Boot를 씁니다. 내부적으로 현재 시스템의 `write_uboot_platform_mtd()`를 호출합니다.

### 19.5 재부팅

USB를 제거하고 재부팅합니다.

```text
SPI NOR U-Boot
  -> autoboot/bootfunc
    -> UFS /boot.scr
      -> UFS armbianEnv.txt
        -> UFS Image/DTB
          -> Linux
```

확인:

```bash
cat /proc/cmdline
cat /proc/mtd
dmesg | grep -iE 'mpxy|ctf|thermal'
systemctl --failed
```

### 19.6 주의

- `armbian-install`은 **현재 부팅한 rootfs를 복사**합니다. 빌드한 `.img` 파일을 직접 쓰는 도구가 아닙니다.
- 현재 실행 중인 UFS를 target으로 지정할 수 없습니다. live USB 부팅이 필요합니다.
- `--target /dev/sda`가 정말 UFS인지 `lsblk`로 확인하세요. USB를 지우면 안 됩니다.
- `/proc/mtd`에 `bootinfo`, `fsbl`, `env`, `esos`, `opensbi`, `uboot`가 있어야 합니다.
- 이 보드에서 `armbian-install` 전체 클린 설치는 아직 실제로 끝까지 검증하지 않았습니다. 현재 검증된 경로는 17장 TitanTools입니다.

### 19.7 실패 시

1. SPI NOR 백업이 있는지 확인합니다.
2. `/var/log/armbian-install.log`를 확인합니다.
3. 공장 SPI NOR만 있으면 17장 TitanTools로 UFS+SPI NOR을 씁니다.
4. 부트로더만 문제면 14.11의 `write_uboot_platform_mtd()` 또는 `flashcp`를 사용합니다.


---

## 마무리

이 문서의 핵심은 다음 한 문장으로 정리할 수 있습니다.

> **SPI NOR에는 부트로더를, UFS에는 운영체제를 둔다. Armbian U-Boot가 UFS의 `boot.scr`을 찾을 수 있게 하고, Linux가 SPI NOR 파티션을 볼 수 있게 하면, Armbian만으로 전체 부팅 체인을 관리할 수 있다.**

이번 작업으로 AIBOX-K3는 다음 상태가 되었습니다.

- 공장 SPI NOR U-Boot 의존 제거
- Armbian이 빌드한 Firefly U-Boot/OpenSBI/ESOS로 SPI NOR 부팅
- U-Boot 패치 `001`~`003` rebase 완료
- Armbian distro boot로 Linux 부팅
  - SPI NOR U-Boot → `autoboot`/`bootfunc` → `/boot.scr` → `armbianEnv.txt` → `Image`/DTB
- CTF2301 팬 제어 정상 (PWM 16~60/255, 온도에 따라 약 550~1,440 RPM)
- CPU 온도 표시 정상 (`thermal_zone0`, 35~39°C)
- SPI NOR 수동 백업/복구 경로 확보
- Firefly 설정은 AIBOX-K3에만 적용
  - 다른 `spacemit-k3` 보드는 기존 `spacemit-com` `k3-br-v1.0.8`과 old patch dir 유지
- Armbian upstream에 올릴 수 있는 보드 설정 완성

이 문서가 같은 보드를 사용하는 다른 사람에게 도움이 되기를 바랍니다.
