# Firefly AIBOX-K3 Armbian 지원 정리 문서

이 문서는 **Armbian upstream에 AIBOX-K3 지원을 기여하기 위한 정리 문서**입니다.
기준 커밋은 `4646b10ded34815cb30893b52a8eca0990b05762` ("Add support for Firefly AIBOX-K3")입니다.

- 대상 보드: Firefly AIBOX-K3 (SpacemiT K3, RISC-V, 8GB RAM, 128GB UFS, 8MiB SPI NOR)
- Armbian: `26.11.0-trunk`, `legacy` branch, kernel `6.18.3-legacy-spacemit-k3`
- U-Boot: `2022.10_armbian-2022.10-Se471-P78cf-Hde31-V0a62-B56a7-R448a`

관련 문서:

- `ARMBIAN-AIBOX-K3-SPI-NOR.ko.md` — 작업 일지, 시행착오, 최종 PR 본문
- `README.md` — 최초에 정리한 메모(질문 목록)

이 문서는 **커밋 `4646b10de`가 무엇을 바꾸는지**, **왜 그렇게 바꿨는지**, **그 결과를 어떻게 빌드하고 플래싱하는지**를 정리한 것입니다.
팬/전원관리 문제는 "펌웨어 소스를 왜 맞췄는가"의 근거로만 짧게 다룹니다.

---

## 목차

0. [이 문서의 사용법](#0-이-문서의-사용법)
1. [커밋 4646b10de가 바꾸는 것](#1-커밋-4646b10de가-바꾸는-것)
2. [하드웨어와 부팅 구조](#2-하드웨어와-부팅-구조)
3. [펌웨어 소스를 왜 Firefly 태그로 맞췄나](#3-펌웨어-소스를-왜-firefly-태그로-맞췄나)
4. [Armbian U-Boot와 boot script](#4-armbian-u-boot와-boot-script)
5. [BSP 온도 수정이 실제로 하는 일](#5-bsp-온도-수정이-실제로-하는-일)
6. [빌드](#6-빌드)
7. [설치](#7-설치)
8. [검증](#8-검증)
9. [Armbian 기여 준비](#9-armbian-기여-준비)
10. [용어집과 명령 모음](#10-용어집과-명령-모음)

---

## 0. 이 문서의 사용법

### 0.1 메모 질문 → 문서 위치

| 메모에 적힌 질문 | 답이 있는 절 |
|---|---|
| 부팅순서 그림이 정말 맞는지, 어디서 아는지 | [2.1](#21-부팅-순서와-각-단계-근거) |
| 파티션 표가 맞는지, `uInitrd`/`dtb`/`boot.scr`가 뭔지 | [2.2](#22-저장-장치-파티션), [2.3](#23-용어-정리) |
| `CTF2301`이 뭔지, 어떻게 확인하나 | [3.2](#32-ctf2301-팬-컨트롤러와-mpxy-메시지) |
| `SBI MPXY extension not available`가 뭔가 | [3.2](#32-ctf2301-팬-컨트롤러와-mpxy-메시지) |
| SDK는 google repo 형식, 태그가 달랐던 이유 | [3.1](#31-sdk와-태그-차이) |
| 왜 Armbian이 전용 U-Boot를 써야 하나 | [4.1](#41-왜-armbian-전용-u-boot가-필요한가) |
| `boot.scr`와 `env_k3.txt`의 차이 | [4.2](#42-bootscr-vs-env_k3txt) |
| `write_uboot_platform_mtd`는 뭔가 | [4.3](#43-write_uboot_platform_mtd) |
| `pre_umount_final_image__firefly_aibox_k3_boot`는 왜 아직 필요하나 | [4.4](#44-pre_umount_final_image__firefly_aibox_k3_boot) |
| Armbian installer로 live USB 부팅/설치 방법 | [7.1](#71-armbian-installer-armbian-표준-방법) |
| 15.6 BSP 온도 수정이 뭔지, 어떻게 동작하나 | [5](#5-bsp-온도-수정이-실제로-하는-일) |
| 커밋이 upstream에 올리기 적절한지 | [9](#9-armbian-기여-준비) |

### 0.2 읽는 순서

- **기여 준비**: 1장 → 9장
- **빌드/설치만**: 6장 → 7장 → 8장
- **하드웨어 이해**: 2장 → 3장 → 4장

---

## 1. 커밋 4646b10de가 바꾸는 것

### 1.1 한 줄 요약

AIBOX-K3는 부트로더를 **SPI NOR 8MiB**에서 읽고, 운영체제를 **UFS 128GB**에서 읽습니다.
Armbian 기본 빌드는 UFS에만 이미지를 쓰기 때문에, 이 보드를 Armbian만으로 관리하려면
**SPI NOR의 부트로더를 Armbian U-Boot로 교체**해야 합니다. 이 커밋은 그 교체에 필요한
설정, 부트 스크립트, 펌웨어 소스, 설치 헬퍼를 한 번에 추가합니다.

### 1.2 파일별 요약

| 파일 | 변경 | 이유 |
|---|---|---|
| `config/boards/firefly-aibox-k3.wip` | `BOOTSCRIPT`/`BOOTENV_FILE`, `mtd-utils`, `mtdparts` 인자, `write_uboot_platform_mtd()`, Firefly 소스 override hook | 이미지 생성과 SPI NOR 갱신, 보드 전용 펌웨어 선택 |
| `config/sources/families/spacemit-k3.conf` | `uboot_custom_postprocess()`에서 ESOS blob 경로를 변수로 선택 (2줄) | 보드마다 다른 ESOS를 쓰면서 family 기본값은 유지 |
| `packages/blobs/riscv64/spacemit-k3/aibox-k3/esos.itb` | 신규 바이너리 (506,499 bytes) | AIBOX-K3는 Firefly 빌드의 ESOS가 필요 |
| `packages/bsp/common/usr/lib/armbian/armbian-hardware-monitor` | AIBOX-K3만 `soctemp`을 `thermal_zone0`으로 | hwmon0이 팬 컨트롤러라 CPU 온도가 아님 |
| `patch/u-boot/legacy/u-boot-spacemit-k3-firefly/` | U-Boot 패치 001~006 | Firefly 태그용 U-Boot 패치 묶음 (001~003은 rebase) |
| `patch/atf/atf-spacemit-k3-firefly/.gitkeep` | 빈 패치 디렉터리 유지 | Firefly OpenSBI에는 family ATF 패치를 적용하지 않음 |

### 1.3 boot script 관련 설정이 왜 필요하나

Armbian U-Boot는 `boot.scr`을 실행하고, `boot.scr`이 `/boot/armbianEnv.txt`를 읽습니다.
두 파일 모두 빌드 시스템이 만들어 줍니다.

- `BOOTSCRIPT="boot-spacemit.cmd:boot.cmd"`
  - 소스: `config/bootscripts/boot-spacemit.cmd`
  - 결과: `/boot/boot.cmd` → `mkimage -T script` → **`/boot/boot.scr`**
  - 처리 위치: `lib/functions/image/partitioning.sh` (mkimage 호출), `lib/functions/bsp/armbian-bsp-cli-deb.sh` (BSP 패키지 등록)
- `BOOTENV_FILE="spacemit.txt"`
  - 소스: `config/bootenv/spacemit.txt` (`verbosity`, `console`, `overlay_prefix`)
  - 결과: 빌드 중 `rootdev`, `rootfstype`, `extraargs`가 추가된 **`/boot/armbianEnv.txt`**

`mtdparts`는 `extraargs`로 들어갑니다. Linux가 SPI NOR 파티션을 보려면 커널 커맨드라인에
`mtdparts=d420c000.spi:...`가 있어야 하기 때문입니다.

```bash
# config/boards/firefly-aibox-k3.wip
function pre_install_kernel_debs__extra_boot_args() {
	run_host_command_logged "echo 'extraargs=net.ifnames=0 mtdparts=${K3_MTDPARTS}' >> '${SDCARD}/boot/armbianEnv.txt'"
}
```

### 1.4 다른 K3 보드에 미치는 영향

`spacemit-k3` family에는 `k3picoitx` 보드도 있습니다. 이 보드에 영향이 가지 않도록
**family 기본값은 그대로 두고**, AIBOX-K3 보드 hook에서만 덮어씁니다.

```bash
# config/boards/firefly-aibox-k3.wip
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

확인한 결과:

| 보드 | BOOTSOURCE | BOOTBRANCH | BOOTPATCHDIR | ESOS |
|---|---|---|---|---|
| `firefly-aibox-k3` | Firefly | `k3/linux6.18_release_v1.0.0a` | `legacy/u-boot-spacemit-k3-firefly` | 506,499 |
| `k3picoitx` | spacemit-com | `k3-br-v1.0.8` | `legacy/u-boot-spacemit-k3` | 450,419 |

---

## 2. 하드웨어와 부팅 구조

### 2.1 부팅 순서와 각 단계 근거

```text
전원 ON
  |
  v
BootROM (SoC 내부, 수정 불가)
  |  SPI NOR offset 0의 bootinfo를 읽음
  v
bootinfo (80 bytes)  ── FSBL의 주소와 크기를 담고 있음
  |
  v
FSBL.bin (U-Boot SPL)  ── DRAM 초기화, 후속 이미지 로드
  |
  +--> OpenSBI (fw_dynamic.itb)  ── M-mode 런타임
  |         |
  |         v
  +--> U-Boot proper (u-boot.itb) ── S-mode 부트로더
            |
            |  UFS의 /boot.scr 실행
            v
          boot.scr  ── armbianEnv.txt를 읽고 Image/uInitrd/dtb를 로드
            |
            v
          Linux kernel + initrd + DTB
            |
            v
          rootfs (UFS)

  ※ ESOS는 위 체인에 직렬로 들어가는 단계가 아니라, 별도 코어(RCU)에서 실행되는
    펌웨어입니다(3.2 근거). 어떤 코드가 언제 RCU를 깨우는지는 SDK 소스
    (FSBL/OpenSBI 쪽)를 봐야 정확히 알 수 있습니다.
```

근거는 모두 빌드 산출물과 커널/sysfs에서 확인할 수 있습니다.

**근거 1. bootinfo는 FSBL의 위치/크기를 담는다**

`packages/blobs/riscv64/spacemit-k3/partition_4M.json`의 주석:

```json
{
  "name": "bootinfo",
  "comment": "contain address and size info of fsbl, MUST not change its position",
  "offset": "0",
  "size": "128K"
}
```

`bootinfo_spinor.bin`은 80바이트입니다.

```bash
$ od -A d -t x1 bootinfo_spinor.bin
0000000 f0 14 07 b0 01 00 01 00 4e 4f 52 46 00 00 00 00
0000016 00 01 00 00 00 00 01 00 00 00 10 00 00 00 00 00
...
```

- offset 8에 `4e 4f 52 46` = `"NORF"` 매직 문자열
- 그 뒤 32비트 워드들에 FSBL 오프셋/크기와 다른 영역 크기가 들어 있습니다
- 정확한 필드 정의는 Firefly SDK 소스가 기준입니다 (`/home/leopark/code/k3-buildroot-sdk`)

**근거 2. OpenSBI가 U-Boot보다 먼저 로드된다**

```bash
$ strings -n 8 fw_dynamic.itb | head -3
Configuration to load OpenSBI before U-Boot
OpenSBI fw_dynamic Firmware
...
$ strings -n 8 u-boot.itb | grep -i 'FIT image'
U-boot FIT image for k3
```

**근거 3. ESOS는 보조(RT) 코어용 펌웨어다**

```bash
$ strings -n 6 esos.itb | head -6
ESOS dual-core image for K3
images
rcpu-data-null
Using data interaction with AP
standalone
#riscv
null.spacemit
```

- `"ESOS dual-core image for K3"`: K3용 듀얼 코어 이미지
- `rcpu0-dtb-com260`, `k3_rt240_com260.dtb`: RT 코어(RCPU)용 DTB
- `"Using data interaction with AP"`: RCU가 AP(=Linux application processor)와 mailbox로 통신

즉 ESOS는 U-Boot 체인에 직렬로 들어가는 단계가 아니라, **별도 코어에서 돌며 Linux와
mailbox로 통신하는 펌웨어**입니다. 그래서 펌웨어 태그가 어긋나면 이 통신 경로가 끊깁니다(3장).

### 2.2 저장 장치 파티션

**SPI NOR (8MiB, `d420c000.spi`)**

| mtd | 이름 | 크기 | offset | 내용 | 근거 |
|---|---|---:|---:|---|---|
| mtd0 | `d420c000.spi` | 8MiB | 0 | master | `/proc/mtd` |
| mtd1 | `bootinfo` | 128KiB | 0 | FSBL 주소/크기 | `partition_4M.json` |
| mtd2 | `fsbl` | 512KiB | 128KiB | U-Boot SPL | `partition_4M.json` |
| mtd3 | `env` | 64KiB | 640KiB | U-Boot 환경변수 | `partition_4M.json` |
| mtd4 | `esos` | 1MiB | 704KiB | ESOS | `partition_4M.json` |
| mtd5 | `opensbi` | 384KiB | 1728KiB | OpenSBI | `partition_4M.json` |
| mtd6 | `uboot` | 나머지(≈5.9MiB) | 2112KiB | U-Boot proper | `mtdparts` in env |

확인 명령:

```bash
cat /proc/mtd
cat /proc/cmdline | tr ' ' '\n' | grep mtdparts
```

U-Boot 쪽 mtdparts(환경변수):

```text
mtdids=nor0=d420c000.spi
mtdparts=d420c000.spi:128K@0(bootinfo),512K@128K(fsbl),64K@640K(env),1M@704K(esos),384K@1728K(opensbi),-@2112K(uboot)
```

주의: `partition_4M.json`은 Firefly 공장 JSON이라 `uboot` 크기가 3M로 적혀 있습니다.
Armbian의 mtdparts는 마지막 파티션을 NOR 끝까지 사용합니다. **오프셋은 같으므로 문제없고,
크기 차이는 NOR 용량 안쪽이라 무해합니다.** U-Boot를 쓸 때는 항상 `mtdparts` 기준을 따릅니다.

**UFS (128GB)**

| 파티션 | 크기 | 파일시스템 | Linux 장치 | 라벨 |
|---|---:|---|---|---|
| p1 | 256MiB | FAT | `/dev/sda1` | `armbi_boot` |
| p2 | 나머지 | ext4 | `/dev/sda2` | `armbi_root` |

확인 명령:

```bash
lsblk -o NAME,SIZE,TYPE,MODEL,TRAN
ls -l /boot
```

`/boot`에 있는 것:

| 파일 | 설명 |
|---|---|
| `Image` | 커널 이미지 (riscv64, uncompressed) |
| `uInitrd` | initrd를 U-Boot legacy image 헤더로 감싼 것. `booti`가 이 형태를 요구 |
| `dtb/spacemit/k3-firefly-aibox-k3.dtb` | 보드 하드웨어 설명 (Device Tree Blob) |
| `boot.scr` | U-Boot 스크립트 이미지 (아래 2.3) |
| `boot.cmd` | `boot.scr`의 원본 텍스트 |
| `armbianEnv.txt` | 커널/DTB/cmdline 설정 |
| `env_k3.txt` | 벤더 U-Boot용 환경 파일 (Armbian 경로에서는 미사용) |

### 2.3 용어 정리

| 용어 | 정체 | 어디서 만들어지나 |
|---|---|---|
| `boot.scr` | U-Boot 스크립트 이미지. `mkimage -T script`로 만든 이진 | `config/bootscripts/boot-spacemit.cmd` → `partitioning.sh` |
| `armbianEnv.txt` | `fdtfile`, `extraargs`, `rootdev`, `rootfstype` 등을 담은 텍스트 | `config/bootenv/spacemit.txt` + 빌드 스크립트 |
| `Image` | 커널 | 커널 빌드 |
| `uInitrd` | U-Boot legacy image로 감싼 initrd | `lib/functions/image/partitioning.sh` |
| `dtb` | Device Tree Blob. 보드의 레지스터/핀/주변기기 정의 | 커널 빌드 |
| `env_k3.txt` | **벤더** U-Boot가 읽는 환경 파일(`knl_name`, `dtb_dir`, `console` 등) | `pre_umount_final_image__firefly_aibox_k3_boot` |
| `mtdparts` | 커널 커맨드라인으로 MTD 파티션 레이아웃을 알려주는 문자열 | `armbianEnv.txt`의 `extraargs` |
| `extlinux` | U-Boot 표준 부팅 설정 형식. `boot.scr` 대신 쓸 수 있음 | 미사용(범위 밖) |

`boot.scr`이 하는 일(요약):

```bash
load ${devtype} ${devnum}:${distro_bootpart} ${scriptaddr} ${prefix}/armbianEnv.txt
env import -t ${scriptaddr} ${filesize}
setenv bootargs "${consoleargs} root=${rootdev} rootfstype=${rootfstype} rw fsck.repair=yes rootwait loglevel=${verbosity} ${extraargs} ${plymouthargs}"
load ... ${initrd}          # uInitrd
load ... ${fk_kvers}        # Image
booti ${kernel_addr_r} ${ramdisk_addr_r}:${filesize} ${fdt_addr_r}
```

### 2.4 확인 명령 모음

```bash
# 파티션
cat /proc/mtd
lsblk -o NAME,SIZE,TYPE,MODEL,TRAN

# 부팅 체인
cat /proc/cmdline
dmesg | grep -iE 'mpxy|mtd|spi|ufs|scsi'

# 부트 파일
ls -l /boot
cat /boot/armbianEnv.txt
strings /boot/boot.scr | head -20

# 펌웨어
cat /sys/class/hwmon/hwmon*/name
for z in /sys/class/thermal/thermal_zone*; do echo "$z $(cat $z/type) $(cat $z/temp)"; done
for c in /sys/class/thermal/cooling_device*; do echo "$c $(cat $c/type) $(cat $c/cur_state)"; done
```

---

## 3. 펌웨어 소스를 왜 Firefly 태그로 맞췄나

### 3.1 SDK와 태그 차이

Firefly AIBOX-K3 펌웨어는 Android 계열의 **repo(manifest)** 방식으로 관리됩니다.
SDK 한 개에 U-Boot, OpenSBI, 커널, 빌드 도구가 각각 별도 git 저장소로 들어 있고,
`manifest.xml`이 버전을 고정합니다. 그래서 "SDK가 어떤 태그를 썼는가"를 보려면
manifest를 확인해야 합니다.

```text
공장 SDK   : Firefly-spacemit-linux/uboot-2022.10  tag k3/linux6.18_release_v1.0.0a
             Firefly-spacemit-linux/opensbi         tag k3/linux6.18_release_v1.0.0a
공장 U-Boot : e471ad131b "riscv: dts: update k3-firefly-common from k3_com260_ifx"
공장 ESOS   : SDK 빌드 산출물 (506,499 bytes)

Armbian 기존: spacemit-com/uboot-2022.10           tag k3-br-v1.0.8
              spacemit-com/opensbi                  tag k3-br-v1.0.8
```

Armbian이 처음에 `spacemit-com`의 오래된 태그를 쓰다가, 이 보드만 **Firefly 태그로
맞춰서** U-Boot/OpenSBI/ESOS를 다시 빌드했습니다. 이유와 근거는 3.2입니다.

### 3.2 CTF2301 팬 컨트롤러와 MPXY 메시지

`spacemit-com` 펌웨어로 SPI NOR을 교체했을 때, Linux가 다음을 출력했습니다.

```text
riscv-sbi-mpxy-mbox soc:mpxy_mbox@0: SBI MPXY extension not available
```

- `riscv-sbi-mpxy-mbox`: 커널의 SBI mailbox 드라이버
- MPXY: OpenSBI(M-mode)와 Linux(S-mode) 사이에서 RCU(RT 코어) mailbox를 중계하는 SBI 확장
- 확장이 없으면 드라이버가 등록되지 않아, RCU와 통신하는 power controller가 probe되지 않고,
  그 아래 i2c 버스(팬 컨트롤러가 붙은 i2c6)도 probe되지 않습니다
- 그 결과 CTF2301 팬 컨트롤러가 아예 없었고, PWM 제어가 되지 않아 팬이 최대로 돌았습니다

Firefly 태그로 맞추면 OpenSBI가 MPXY를 제공하고, 모든 probe가 정상화됩니다.

```text
riscv-sbi-mpxy-mbox soc:mpxy_mbox@0: mailbox registered with 5 channels
```

팬 컨트롤러는 아래처럼 확인합니다.

```bash
$ cat /sys/class/hwmon/hwmon0/name
ctf2301

$ for c in /sys/class/thermal/cooling_device*; do echo "$c $(cat $c/type)"; done
/sys/class/thermal/cooling_device0 ctf2301_fan

$ cat /sys/class/hwmon/hwmon0/temp1_input
345625
```

- `hwmon0`의 이름이 `ctf2301` → hwmon0는 **팬 컨트롤러**입니다
- `cooling_device0`의 type이 `ctf2301_fan` → PWM 제어가 이 디바이스로 들어갑니다
- `temp1_input = 345625`는 CPU 온도가 아니라 컨트롤러 내부 센서 값이며,
  SoC thermal zone과 스케일(10배)이 다릅니다. SoC 온도는 `thermal_zone0/temp`입니다

이 차이가 5장의 BSP 수정으로 이어집니다.

---

## 4. Armbian U-Boot와 boot script

### 4.1 왜 Armbian 전용 U-Boot가 필요한가

메인라인 U-Boot(`u-boot/u-boot`)에는 SpacemiT K3 보드와 `k3_defconfig`가 없습니다.
그래서 Armbian은 K1/K3 family처럼 **벤더 소스 + Armbian 패치**로 U-Boot를 만듭니다.
`spacemit-k3` family가 원래 그래 왔고, 이 보드는 Firefly 소스/태그를 쓰도록 바뀐 것입니다.

Armbian이 U-Boot를 직접 빌드해서 얻는 이득:

1. 부트 체인 전체를 재현 가능하게 관리 (SPI NOR 6개 파티션 파일이 deb로 나옴)
2. Armbian 표준 boot script(`boot.scr` + `armbianEnv.txt`)와 연동
3. `armbian-install`이 쓸 수 있는 설치 함수(`write_uboot_platform_mtd`)를 패키지에 포함
4. Armbian 커널/디바이스 트리와 일치하는 설정(defconfig, env)

산출물:

```text
output/debs/linux-u-boot-firefly-aibox-k3-legacy_...deb
  └── /usr/lib/u-boot/platform_install.sh   ← armbian-install이 sourcing 하는 헬퍼
  └── /usr/lib/linux-u-boot-legacy-firefly-aibox-k3/
        ├── bootinfo_spinor.bin   (80)
        ├── FSBL.bin              (453,984)
        ├── u-boot-env-default.bin(16,384)
        ├── esos.itb              (506,499)
        ├── fw_dynamic.itb        (271,311)
        ├── u-boot.itb            (2,683,538)
        ├── partition_4M.json
        └── spi-flash-howto.txt
```

### 4.2 boot.scr vs env_k3.txt

| 항목 | `boot.scr` (+ `armbianEnv.txt`) | `env_k3.txt` |
|---|---|---|
| 만든이 | Armbian 빌드 시스템 | Armbian 빌드 시스템이 만드는 **벤더 U-Boot용** 보조 파일 |
| 읽는 주체 | Armbian U-Boot | Firefly(공장) U-Boot의 `k3.c` |
| 내용 | U-Boot 스크립트. `armbianEnv.txt`를 읽어 Image/uInitrd/dtb 로드 | `console`, `bootdelay`, `knl_name=Image`, `dtb_dir=dtb/spacemit`, `loglevel` |
| Armbian 경로에서 사용? | **사용함** (최종 부팅 경로) | 사용하지 않음 (공장 U-Boot 1차 부팅용) |

U-Boot env는 U-Boot가 저장하는 환경변수이고, `env_k3.txt`는 그 **텍스트 사본**입니다.
공장 U-Boot는 `nor_boot` → `set_nor_args` → `boot_kernel` 체인으로 부팅하며,
이 체인이 `knl_name`/`dtb_dir`을 `env_k3.txt`에서 읽습니다.

Armbian U-Boot는 패치 `003`으로 env를 `autoboot`/`bootfunc` 체인으로 교체했습니다.

```text
autoboot= ... setenv devtype usb; usb start; sleep 2; run bootfunc;
           setenv devtype mmc; run bootfunc;
           setenv devtype nvme; pci enum; nvme scan; nvme part; run bootfunc;
           setenv devtype scsi; run bootfunc; ...
bootcmd=run autoboot;
```

`bootfunc`는 `extlinux.conf` → `/boot.scr` → `/boot/boot.scr` → EFI 순서로 찾습니다.
그래서 USB/UFS 어디에 이미지가 있든 같은 경로로 부팅합니다.

### 4.3 write_uboot_platform_mtd

`armbian-install`(=`armbian-config --api module_partitioner`)이 SPI/MTD 부트로더를 쓸 때
호출하는 함수입니다. 계약(armbian-config 소스 기준):

```text
write_uboot_platform_mtd <dir> <device> <log> <mtd_list>
```

- `<dir>`: U-Boot 바이너리가 있는 디렉터리
- `<device>`: 첫 번째 대상 장치
- `<log>`: 로그 파일
- `<mtd_list>`: 설치 대상 목록 (`mtdblockN` 또는 `mtdN:label`)

이번 보드 구현은 목록을 그대로 쓰지 않고, **파티션 이름으로 `/proc/mtd`에서 찾아** 6개를
모두 씁니다. K3는 NOR 전체가 `mtd0`으로 보이고 실제 파티션이 `mtd1..6`이 되며,
`CONFIG_MTD_PARTITIONED_MASTER=y` 때문에 K1 보드와 번호가 다릅니다.

```bash
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
```

- `flashcp`는 `mtd-utils` 패키지가 제공합니다 → `PACKAGE_LIST_BOARD="mtd-utils"`
- **자동 백업을 하지 않습니다.** 쓰기 전에 수동으로 백업하세요(7.3)
- 함수는 `config/boards/firefly-aibox-k3.wip`에 있고, U-Boot deb의
  `platform_install.sh`에 그대로 복사되어 들어갑니다
  (`lib/functions/compilation/uboot.sh`가 `declare -f`로 뽑아 씁니다)

### 4.4 pre_umount_final_image__firefly_aibox_k3_boot

이미지 빌드 마지막에 `/boot/env_k3.txt`를 만들어 넣는 hook입니다.

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

**왜 아직 있나**

- TitanTools 클린 설치는 "UFS 먼저 쓰고, SPI NOR은 나중에" 진행합니다.
- 그 사이 공장 U-Boot로 부팅해야 하는데, 공장 U-Boot는 `env_k3.txt`가 없으면
  Armbian 이미지의 커널/DTB 위치를 알 수 없습니다.
- 즉 **공장 SPI NOR로 1차 부팅**하는 경로에서만 필요합니다.

**언제 필요 없나**

- `armbian-install --boot mtd`로 SPI NOR을 먼저 Armbian U-Boot로 쓴다면 불필요합니다.
- Armbian U-Boot는 `env_k3.txt`를 읽지 않습니다.

**upstream 판단**

- 지금은 유지해도 부팅에 영향이 없고(6줄), "UFS만 먼저 쓰는 설치"를 가능하게 해 줍니다.
- 다만 upstream 리뷰는 "dead file"을 지적할 수 있습니다.
- 정리 방향: 유지하되 PR 본문에 "factory U-Boot first boot" 용도로 명시하거나,
  `armbian-install` 경로가 기본이 되면 제거하는 두 가지 중 하나를 고릅니다.

---

## 5. BSP 온도 수정이 실제로 하는 일

수정 파일: `packages/bsp/common/usr/lib/armbian/armbian-hardware-monitor`

```diff
-	# SoC temp: check sun7i legacy exception first, then standard path(s)
-	if [[ -d "/sys/devices/platform/a20-tp-hwmon" ]]; then
+	# SoC temp: check board-specific exceptions first, then standard path(s)
+	if [[ ${BOARD} == "firefly-aibox-k3" ]]; then
+		# CTF2301 (hwmon0) is the fan controller, not the SoC sensor.
+		ln -fs /sys/devices/virtual/thermal/thermal_zone0/temp /etc/armbianmonitor/datasources/soctemp
+	elif [[ -d "/sys/devices/platform/a20-tp-hwmon" ]]; then
```

### 5.1 동작 구조

1. `armbian-hardware-monitor`의 `prepare_temp_monitoring()`이 부팅 시 실행됩니다.
2. 이 함수는 `/etc/armbianmonitor/datasources/` 아래에 **심링크**를 만듭니다.
   - `soctemp` → SoC 온도
   - `pmictemp` → PMIC 온도
3. `armbianmonitor`(MOTD/모니터링)가 이 심링크를 읽어서 "CPU temp: 39°C"처럼 표시합니다.

```text
/etc/armbianmonitor/datasources/soctemp
   -> /sys/devices/virtual/thermal/thermal_zone0/temp
```

### 5.2 왜 보드별 분기가 필요한가

기본 규칙은 `hwmon0/temp1_input`입니다. AIBOX-K3에서 `hwmon0`은 CTF2301 팬 컨트롤러라서

```text
/sys/class/hwmon/hwmon0/temp1_input = 345625   # 10배 스케일, SoC 온도 아님
/sys/class/thermal/thermal_zone0/temp = 35000   # 실제 SoC 온도(35°C)
```

기본 규칙을 따르면 MOTD에 345°C가 표시됩니다. 그래서 AIBOX-K3만 `thermal_zone0`을 씁니다.

### 5.3 검증

```bash
readlink -f /etc/armbianmonitor/datasources/soctemp
cat /etc/armbianmonitor/datasources/soctemp      # 39000 (39°C)
cat /sys/class/hwmon/hwmon0/temp1_input           # 345625 (팬 컨트롤러 내부 센서)
```

같은 파일에 `helios4` 보드 분기가 이미 있으므로 upstream에서도 precedent가 있습니다.

### 5.4 남는 문제

CTF2301 드라이버의 온도 스케일(10배) 자체는 그대로입니다.
BSP는 표시에만 `thermal_zone0`을 씁니다. 드라이버 수정은 별도 과제입니다.

---

## 6. 빌드

### 6.1 환경

- x86_64 Ubuntu 호스트에서 cross-compile (RISC-V 보드에서 빌드하지 않음)
- Docker/Podman, 16코어/16GB 이상 권장

```bash
cd ~/code/armbian/build
sudo ./compile.sh requirements      # 처음 한 번만
```

### 6.2 명령

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
| `KERNEL_CONFIGURE=no` | kernel config 확인 생략 |
| `RELEASE=resolute` | Ubuntu 26.04 기반 |

### 6.3 산출물과 용도

```text
output/images/Armbian-unofficial_26.11.0-trunk_Firefly-aibox-k3_resolute_legacy_6.18.3.img
output/images/...img.sha
output/images/...img.txt

output/debs/linux-u-boot-firefly-aibox-k3-legacy_26.11.0-trunk_riscv64__2022.10-Se471-P78cf-Hde31-V0a62-B56a7-R448a.deb
```

| 산출물 | goes to | 설명 |
|---|---|---|
| `*.img` | UFS 전체 또는 USB | GPT + bootfs(256MiB FAT) + rootfs(ext4). UFS에 쓰면 UFS가 완성됩니다 |
| `*.img` 안의 `/boot` | UFS bootfs | `boot.scr`, `armbianEnv.txt`, `Image`, `uInitrd`, `dtb/`, `env_k3.txt` |
| `linux-u-boot-*.deb` | **SPI NOR** | `bootinfo`/`fsbl`/`env`/`esos`/`opensbi`/`uboot` 6개 파일과 `platform_install.sh` |
| `linux-u-boot-*.deb` | 이미지 안에도 포함 | `armbian-install`이 실행 중인 시스템에서 이 deb의 `platform_install.sh`를 사용합니다 |

이미지를 UFS에 썼다고 SPI NOR이 바뀌지는 않습니다. **두 곳을 따로 써야 합니다.**

`compile.sh`는 TitanTools용 zip을 만들지 않습니다. zip이 필요하면
`ARMBIAN-AIBOX-K3-SPI-NOR.ko.md` 17.3의 조립 절차를 사용하세요.

### 6.4 빌드 결과 확인

```bash
IMG=$(ls -t output/images/*.img | head -1)
cat "${IMG}.sha"

DEB=$(ls -t output/debs/linux-u-boot-firefly-aibox-k3-legacy_*.deb | head -1)
rm -rf /tmp/uboot-check && mkdir -p /tmp/uboot-check
dpkg-deb -x "$DEB" /tmp/uboot-check
grep -n write_uboot_platform_mtd /tmp/uboot-check/usr/lib/u-boot/platform_install.sh
ls -l /tmp/uboot-check/usr/lib/linux-u-boot-legacy-firefly-aibox-k3/
```

이미지 안쪽 확인:

```bash
sudo losetup -b 4096 -fP --show "$IMG"
sudo mount -o ro /dev/loop22p1 /mnt/aibox-boot
ls -l /mnt/aibox-boot
cat /mnt/aibox-boot/armbianEnv.txt
sudo mount -o ro /dev/loop22p2 /mnt/aibox-root
grep -n write_uboot_platform_mtd /mnt/aibox-root/usr/lib/u-boot/platform_install.sh
sudo umount /mnt/aibox-boot /mnt/aibox-root && sudo losetup -d /dev/loop22
```

---

## 7. 설치

AIBOX-K3는 **SPI NOR(부트로더)**과 **UFS(OS)** 두 곳을 써야 합니다.

| 방법 | UFS | SPI NOR | 상태 |
|---|---|---|---|
| Armbian installer (`armbian-install --boot mtd`) | 자동 | 자동 | Armbian 표준. 이 보드에서 end-to-end 검증은 아직 없음 |
| TitanTools | `partition_universal.json` | `partition_4M.json` | **실제로 검증된 경로** |
| 수동 | `dd` | `flashcp` | 최소 도구 |

### 7.1 Armbian installer (Armbian 표준 방법)

`armbian-install`은 UFS/MTD/NVMe/eMMC/USB 타깃을 지원합니다.
`--boot mtd`를 주면 SPI NOR에 U-Boot를 쓰고, `--target`으로 지정한 디스크에 rootfs를 설치합니다.

#### 7.1.1 전제

- SPI NOR에 **Armbian U-Boot가 이미** 있어야 합니다.
  - 공장 SPI NOR만 있으면 USB 부팅이 안 될 수 있습니다 → 먼저 TitanTools로 SPI NOR을 쓰세요
- Armbian 이미지를 담은 **live USB**가 필요합니다
- 설치 대상 UFS는 `/dev/sda`로 보입니다 (`lsblk`로 반드시 확인)
- **쓰기 전에 SPI NOR 백업** (7.3)

#### 7.1.2 live USB 만들기

```bash
IMG=$(ls -t output/images/*.img | head -1)
lsblk -o NAME,SIZE,TYPE,MODEL,TRAN          # 대상 USB 확인

sudo dd if="$IMG" of=/dev/sdX bs=4M status=progress conv=fsync
```

`/dev/sdX`는 **USB** 장치입니다. AIBOX의 UFS(`/dev/sda`)를 지우면 안 됩니다.

#### 7.1.3 USB로 부팅

1. AIBOX 전원을 끄고 USB를 연결합니다.
   - USB 저장장치로 잡히는 포트를 사용합니다(OTG Type-C 또는 USB-A)
2. 전원을 켭니다.
3. 시리얼 콘솔(115200)에서 확인:

```text
U-Boot 2022.10_armbian-2022.10-Se471-P78cf-Hde31-V0a62-B56a7-R448a
Loading K3 environment
Loading Armbian environment (usb 0:1)
Booting Armbian (usb 0:1)
Starting kernel ...
```

`usb 0:1`이 보이면 USB 부팅 성공입니다. `usb start`가 장치를 못 잡으면
U-Boot 프롬프트에서 `usb start` → `usb list`로 확인하세요.

주의:

- U-Boot env의 `autoboot`는 **USB를 가장 먼저** 스캔합니다. USB가 꽂혀 있으면 UFS보다 USB가 우선입니다.
- `bootdelay=2`라서 2초 안에 아무 키나 눌러 U-Boot 프롬프트로 들어갈 수 있습니다.
  프롬프트에서 `usb start`로 장치를 직접 확인할 수 있습니다.

4. 부팅 후 어디에서 실행 중인지 확인:

```bash
findmnt -no SOURCE /
lsblk -o NAME,SIZE,TYPE,MODEL,MOUNTPOINTS
```

`/`가 USB 파티션이고, UFS가 `/dev/sda`로 보여야 합니다.

#### 7.1.4 설치

대화형:

```bash
sudo armbian-install
```

- target: UFS (`/dev/sda`)
- boot: `SPI/MTD`
- filesystem: `ext4`

비대화형:

```bash
sudo armbian-install --target /dev/sda --boot mtd --fs ext4 --yes
```

내부 동작:

- 실행 중인 rootfs를 target으로 복사합니다
- `--boot mtd`면 `/proc/mtd`에서 대상 목록을 만들어
  `write_uboot_platform_mtd`를 호출해 SPI NOR을 씁니다
- 로그: `/var/log/armbian-install.log`

#### 7.1.5 재부팅

USB를 뽑고 재부팅합니다.

```text
SPI NOR U-Boot
  -> autoboot/bootfunc
    -> UFS /boot.scr
      -> UFS armbianEnv.txt
        -> UFS Image/DTB
          -> Linux
```

#### 7.1.6 주의

- `armbian-install`은 **현재 부팅한 rootfs를 복사**합니다. 빌드한 `.img`를 직접 쓰는 도구가 아닙니다
- 현재 실행 중인 디스크는 target이 될 수 없습니다 → live USB가 필요합니다
- `--target /dev/sda`가 정말 UFS인지 `lsblk`로 확인하세요
- `/proc/mtd`에 `bootinfo`, `fsbl`, `env`, `esos`, `opensbi`, `uboot`가 있어야 합니다
- **이 보드에서 전체 클린 설치를 끝까지 검증하지는 않았습니다.**
  실패하면 TitanTools 경로(7.2)를 사용하세요

### 7.2 TitanTools (검증된 경로)

패키지: `armbian-firefly-aibox-k3-clean.zip` (17.3 조립 절차 참고)

1. UFS 쓰기
   - 전원 분리 → 플래싱 USB-C 연결 → Boot 버튼 누른 채 전원 연결
   - TitanTools → `Dev Tools → USB Download → Scan Devices`
   - `Configure Partition File`에서 `partition_universal.json`
   - `bootfs.img`, `rootfs.ext4` 배치 → `Start Flashing`
2. SPI NOR 쓰기
   - 같은 절차에서 `partition_4M.json` 선택
   - `bootinfo_spinor.bin`, `FSBL.bin`, `u-boot-env-default.bin`,
     `esos.itb`, `fw_dynamic.itb`, `u-boot.itb` 배치 → `Start Flashing`
3. 재부팅

UFS를 먼저 쓰고 공장 SPI NOR로 1차 부팅할 수 있습니다(이때 `env_k3.txt`가 쓰입니다).

### 7.3 수동

이미지(UFS):

```bash
sudo dd if=output/images/Armbian-unofficial_...img of=/dev/sda bs=4M status=progress conv=fsync
```

U-Boot(SPI NOR) — 반드시 먼저 백업:

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

```bash
sudo dpkg -i linux-u-boot-firefly-aibox-k3-legacy_*.deb
sudo bash -c '
  source /usr/lib/u-boot/platform_install.sh
  write_uboot_platform_mtd "$DIR" /dev/mtd0 /tmp/spinor.log
'
cat /tmp/spinor.log
```

`write_uboot_platform_mtd()`는 **자동 백업을 하지 않습니다.**

### 7.4 복구

1. 전원 분리 → 플래싱 USB-C 연결 → Boot 버튼 누른 채 전원 연결
2. TitanTools로 공장 펌웨어 플래싱, 또는 `spi-flash-howto.txt`의 fastboot 순서

```text
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

`erasespi`는 4MiB만 지웁니다. 전체 NOR 지우기가 아닙니다.

---

## 8. 검증

최종 빌드 기준 실제 확인 결과입니다.

```text
U-Boot SPL 2022.10_armbian-2022.10-Se471-P78cf-Hde31-V0a62-B56a7-R448a
U-Boot 2022.10_armbian-2022.10-Se471-P78cf-Hde31-V0a62-B56a7-R448a
Loading K3 environment
Loading Armbian environment (scsi 0:1)
Booting Armbian (scsi 0:1)
Starting kernel ...
```

| 항목 | 결과 | 확인 명령 |
|---|---|---|
| 부팅 경로 | UFS `boot.scr` → `armbianEnv.txt` | 시리얼 로그, `cat /proc/cmdline` |
| SPI NOR 6개 파티션 | 전부 MATCH | `cmp -n <size> <file> /dev/mtdN` |
| MPXY | `mailbox registered with 5 channels` | `dmesg \| grep mpxy` |
| 팬 | `ctf2301_fan`, PWM 60/255, 1,444 RPM | `cat /sys/class/thermal/cooling_device0/type` |
| CPU 온도 | `thermal_zone0` 39°C | `cat /sys/class/thermal/thermal_zone0/temp` |
| 시스템 | `systemctl --failed` 0 | `systemctl --failed` |
| 다른 K3 보드 | `k3picoitx`가 기존 소스/패치/ESOS 사용 | U-Boot 빌드 로그 |

`/proc/cmdline`:

```text
console=tty0 console=ttyS0,115200n8 root=PARTUUID=... rootfstype=ext4 rw fsck.repair=yes
rootwait loglevel=1 net.ifnames=0 mtdparts=d420c000.spi:128K@0(bootinfo),...,-@2112K(uboot)
```

---

## 9. Armbian 기여 준비

### 9.1 그대로 두어도 되는 것

| 항목 | 판단 | 이유 |
|---|---|---|
| `post_family_config__...` hook에서 Firefly 소스 지정 | 유지 | 다른 K3 보드에 영향 없음. `k3picoitx` 빌드로 확인 |
| family의 `K3_ESOS_BLOB` 한 줄 | 유지 | 기본값이 그대로여서 다른 보드 동작 불변 |
| `packages/bsp/common`의 `${BOARD}` 분기 | 유지 | 같은 파일에 `helios4` precedent 있음 |
| U-Boot 패치 001~003 rebase | 유지 | Armbian 표준 distro boot 경로의 핵심 |
| U-Boot 패치 004~006 | 유지 | upstream K3 패치. 004=EFI 부팅, 005=로그, 006=빌드 경고 |
| `mtd-utils`, `mtdparts`, `write_uboot_platform_mtd()` | 유지 | SPI NOR 부트로더 관리의 핵심 |

### 9.2 upstream 제출 전에 정리할 것

| 항목 | 현재 | 권장 | 비고 |
|---|---|---|---|
| 보드 파일 확장자 | `.wip` | `.conf` | upstream은 `.conf`만 받습니다 (`.wip` = work in progress) |
| `BOARD_MAINTAINER` | 없음 | GitHub 아이디 추가 | 필수 |
| `INTRODUCED` | 없음 | 출시 연도 추가 | 다른 보드 관례 |
| `BOARD_NAME` | `AIBOX-K3` | `Firefly AIBOX-K3` | upstream은 "Company Product Version" 규칙 |
| U-Boot 패치 헤더 | "Rebased onto k3-br-v1.0.8" | Firefly 태그 기준임을 표기 | **패치 해시와 U-Boot 버전 문자열이 바뀝니다 → 재빌드 필요** |
| `write_uboot_platform_mtd()` | `$INSTALL_MTD_LIST` 무시 | "보드 전체 NOR" 규칙임을 주석으로 명시 (완료) | armbian-install 계약 |
| 빈 ATF 패치 디렉터리 | `.gitkeep` | `.gitkeep` 유지 | `README.md`로 바꾸면 U-Boot artifact 해시가 바뀌어 산출물을 다시 만들어야 합니다 (아래 참고) |
| `env_k3.txt` 생성 hook | 유지 | 유지 이유를 PR 본문에 명시하거나 제거 | 4.4 참고 |
| `overlay_prefix=k1` | 공유 bootenv 사용 | 무해하지만 K3 전용 bootenv 고려 | overlay를 쓰지 않으면 영향 없음 |

#### U-Boot 버전 문자열에 주의

U-Boot deb의 버전 문자열은 각 요소의 해시로 만들어집니다
(`lib/functions/artifacts/artifact-uboot.sh`).

```text
2022.10 -Se471 -P78cf -Hde31 -V0a62 -B56a7 -R448a
          │       │       │       │       │      └ reversioning
          │       │       │       │       └ framework bash hash (lib/functions/compilation/uboot*.sh)
          │       │       │       └ config 변수 hash (BOOTCONFIG, UBOOT_TARGET_MAP, ATFPATCHDIR ...)
          │       │       └ extension hook + U-Boot 함수 hash
          │       └ U-Boot 패치 hash (BOOTPATCHDIR + ATF 패치 디렉터리)
          └ U-Boot 소스 커밋
```

따라서 아래 중 하나라도 바꾸면 버전 문자열이 바뀝니다.

- `patch/u-boot/...` 또는 `patch/atf/...` 안의 파일 (내용뿐 아니라 **추가/삭제**도)
- `packages/blobs/...`의 ESOS
- `BOOTCONFIG`, `UBOOT_TARGET_MAP`, `ATFPATCHDIR` 등 해시 대상 변수
- build framework 자체(`lib/functions/compilation/uboot*.sh`)

실측 예 1 (ATF 패치 디렉터리만 변경, 소스 동일):

```text
ATF 패치 디렉터리 = .gitkeep (빈 파일)   -> P78cf
ATF 패치 디렉터리 = README.md (내용 있음) -> P3c58
```

실측 예 2 (같은 커밋, build host clone의 upstream base만 다름):

```text
로컬 repo (upstream 023e98c93) -> ...-V0a62-B56a7-R448a
빌드 호스트 (upstream d399a9f87) -> ...-V0a62-Bb936-R448a
```

두 번째 경우는 `B`(framework bash hash)만 다르고, 실제 바이너리 차이는
**u-boot.itb 25바이트(삽입된 버전 문자열)**, FSBL은 FIT 헤더 타임스탬프까지 포함해 273바이트입니다.
기능 차이는 없습니다. 다만 산출물을 비교할 때 이 점을 알아야 합니다.

즉 사소한 정리라도 **deb → 이미지 → 클린 패키지 → 문서 SHA**까지 다시 만들어야 합니다.
PR 제출 전에 정리할 항목이 여러 개이므로, 그 시점에 한 번에 정리하고 전체를 다시 빌드하는 편이 낫습니다.

### 9.3 커밋 구성

upstream PR은 보통 3개 커밋로 나눕니다.

```text
1. board: firefly-aibox-k3: add SPI NOR bootloader update
   - config/boards/firefly-aibox-k3.wip
   - BOOTSCRIPT / BOOTENV_FILE / mtdparts / mtd-utils / write_uboot_platform_mtd()

2. spacemit-k3: use Firefly firmware tag and distro boot
   - config/boards/firefly-aibox-k3.wip (post_family_config hook)
   - packages/blobs/riscv64/spacemit-k3/aibox-k3/esos.itb
   - patch/u-boot/legacy/u-boot-spacemit-k3-firefly/
   - patch/atf/atf-spacemit-k3-firefly/
   - config/sources/families/spacemit-k3.conf (ESOS 선택 한 줄)

3. bsp: firefly-aibox-k3: use SoC thermal zone for CPU temp
   - packages/bsp/common/usr/lib/armbian/armbian-hardware-monitor
```

PR 본문과 테스트 절차는 `ARMBIAN-AIBOX-K3-SPI-NOR.ko.md` 10.3(최종 PR 본문)을 사용하세요.

### 9.4 체크리스트

- [x] AIBOX-K3 이미지 빌드 성공
- [x] SPI NOR 6개 파티션 MATCH
- [x] `boot.scr` distro boot 확인
- [x] MPXY 정상
- [x] 팬 제어 정상
- [x] CPU 온도 표시 정상
- [x] `systemctl --failed` 0
- [x] `k3picoitx` 빌드가 기존 설정 유지 확인
- [ ] `.wip` → `.conf`, `BOARD_MAINTAINER`, `INTRODUCED`, `BOARD_NAME` 정리
- [ ] U-Boot 패치 헤더 표기 정리 후 재빌드
- [ ] `armbian-install --boot mtd` end-to-end 검증
- [ ] upstream PR 제출

### 9.5 커밋만으로 빌드되는지 확인

커밋 `4646b10de`이 필요한 파일을 모두 담고 있는지, `.git` 없이 커밋만으로 빌드되는지 확인했습니다.

```bash
# 커밋 트리만 추출 (.git 없음)
git archive --format=tar.gz -o /tmp/commit.tar.gz 4646b10de
scp /tmp/commit.tar.gz leopark@<build-host>:/tmp/

# 빌드 호스트에서
mkdir -p build-verify && tar -xzf /tmp/commit.tar.gz -C build-verify
ln -s ../build/cache build-verify/cache      # 소스 캐시만 재사용
cd build-verify && ./compile.sh uboot BOARD=firefly-aibox-k3 BRANCH=legacy
```

결과:

```text
[🌱] Using U-Boot patch dir: [ legacy/u-boot-spacemit-k3-firefly ]
[🌱] firefly-aibox-k3 [ Adding ESOS blob ]
[🌱] Built u-boot deb OK [ ... 2022.10-Se471-P78cf-Hde31-V0a62-Bb936-R448a ]
```

- 패치 해시 `P78cf`가 배포된 빌드와 동일 → 커밋 내용이 동일함을 확인
- `B`만 다름: build host clone의 upstream base가 로컬 repo보다 한참 이전
  (`lib/functions/compilation/uboot.sh` 내용 차이). 실제 바이너리 차이는
  u-boot.itb 25바이트(버전 문자열)뿐
- 커밋 트리에 family용 기존 패치/ATF 패치가 그대로 남아 있어, 다른 K3 보드 빌드에 영향 없음

즉 **이 커밋은 단독으로 빌드·검증 가능한 상태**입니다.

---

## 10. 용어집과 명령 모음

### 10.1 용어

| 용어 | 설명 |
|---|---|
| SPI NOR | 부트로더를 담는 8MiB NOR 플래시. BootROM이 여기서 읽습니다 |
| UFS | 128GB 저장장치. bootfs( FAT )와 rootfs( ext4 )로 나눕니다 |
| bootinfo | NOR offset 0의 80바이트 헤더. FSBL 주소/크기 |
| FSBL / SPL | First Stage Boot Loader. DRAM 초기화와 후속 이미지 로드 |
| ESOS | K3 보조(RT) 코어용 펌웨어. AP(Linux)와 mailbox로 통신 |
| OpenSBI | RISC-V M-mode 런타임. S-mode(U-Boot/Linux)로 넘어갑니다 |
| U-Boot proper | S-mode 부트로더. `boot.scr` 또는 `env_k3.txt`를 실행 |
| `boot.scr` | U-Boot 스크립트 이미지. Armbian 표준 부팅 경로 |
| `armbianEnv.txt` | `fdtfile`, `extraargs`, `rootdev` 등 부팅 설정 |
| `env_k3.txt` | 벤더 U-Boot용 환경 파일 |
| `mtdparts` | 커널 커맨드라인으로 NOR 파티션 레이아웃을 알려주는 문자열 |
| ITB / FIT | 여러 이미지를 담은 DTB 컨테이너. `u-boot.itb`, `fw_dynamic.itb`, `esos.itb` |
| MPXY | OpenSBI가 제공하는 SBI 확장. RCU mailbox 중계에 사용 |
| CTF2301 | AIBOX-K3의 PWM 팬 컨트롤러. `hwmon0`으로 보임 |

### 10.2 자주 쓰는 명령

```bash
# 빌드
./compile.sh build BOARD=firefly-aibox-k3 BRANCH=legacy \
  BUILD_DESKTOP=no BUILD_MINIMAL=no EXPERT=yes KERNEL_CONFIGURE=no RELEASE=resolute
./compile.sh uboot BOARD=firefly-aibox-k3 BRANCH=legacy

# 이미지/패키지 확인
ls -t output/images/*.img; cat output/images/*.img.sha
ls -t output/debs/linux-u-boot-firefly-aibox-k3-legacy_*.deb
dpkg-deb -x <deb> /tmp/uboot-check && ls -l /tmp/uboot-check/usr/lib/linux-u-boot-legacy-firefly-aibox-k3/

# 보드 상태
cat /proc/cmdline
cat /proc/mtd
lsblk -o NAME,SIZE,TYPE,MODEL,TRAN
ls -l /boot
dmesg | grep -iE 'mpxy|ctf2301|mtd'

# SPI NOR
sudo bash -c 'source /usr/lib/u-boot/platform_install.sh; write_uboot_platform_mtd "$DIR" /dev/mtd0 /tmp/spinor.log'
cat /sys/class/hwmon/hwmon0/name
cat /sys/class/thermal/thermal_zone0/temp
```

### 10.3 참고 자료

- Armbian build framework: <https://github.com/armbian/build>
- Firefly AIBOX-K3 wiki: <https://wiki.t-firefly.com/en/AIBOX-K3/>
- Firefly AIBOX-K3 UART debug: <https://community.t-firefly.com/en/docs/products/computers/AIBOX-K3/debug>
- Firefly firmware upgrade: <https://wiki.t-firefly.com/en/AIBOX-K3/upgrade_firmware.html>
- SpacemiT K3 U-Boot: <https://github.com/Firefly-spacemit-linux/uboot-2022.10>
- SpacemiT K3 OpenSBI: <https://github.com/Firefly-spacemit-linux/opensbi>
- SpacemiT K3 Linux: <https://github.com/spacemit-com/linux-6.18>
- K3 boot flow 문서: <https://github.com/ganboing/K3-Docs>
- Armbian 보드 설정 문서: `config/boards/README.md`
