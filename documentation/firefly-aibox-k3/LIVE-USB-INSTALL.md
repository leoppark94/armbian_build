# Firefly AIBOX-K3: Armbian만으로 live USB 부팅과 전체 설치

이 문서는 **TitanTools 같은 벤더 도구 없이, Armbian이 만든 파일만으로**
AIBOX-K3에 Armbian을 설치하는 방법을 설명합니다.

- 대상 보드: Firefly AIBOX-K3 (SpacemiT K3, 128GB UFS + 8MiB SPI NOR)
- Armbian: `26.11.0-trunk`, `legacy`, kernel `6.18.3-legacy-spacemit-k3`
- U-Boot: `2022.10_armbian-2022.10-Se471-P78cf-Hde31-V0a62-B56a7-R448a`
- 기준 커밋: `4646b10de` ("Add support for Firefly AIBOX-K3")

관련 문서:

- `AIBOX-K3-GUIDE.md` — 커밋이 무엇을 바꾸는지, 부팅 구조, 용어
- `ARMBIAN-AIBOX-K3-SPI-NOR.ko.md` — 작업 일지와 최종 PR 본문

---

## 목차

0. [요약: 어떤 경로를 쓸 것인가](#0-요약-어떤-경로를-쓸-것인가)
1. [사전 준비물](#1-사전-준비물)
2. [산출물 만들기](#2-산출물-만들기)
3. [live USB 만들기](#3-live-usb-만들기)
4. [경로 A: Armbian U-Boot가 이미 있는 보드](#4-경로-a-armbian-u-boot가-이미-있는-보드)
5. [경로 B: 공장 상태에서 시작 (fastboot)](#5-경로-b-공장-상태에서-시작-fastboot)
6. [경로 C: 공장 U-Boot의 "recovery from usb" 사용](#6-경로-c-공장-u-boot의-recovery-from-usb-사용)
7. [armbian-install 실행](#7-armbian-install-실행)
8. [재부팅 후 검증](#8-재부팅-후-검증)
9. [복구](#9-복구)
10. [부록](#10-부록)

---

## 0. 요약: 어떤 경로를 쓸 것인가

이 보드는 부트로더가 **SPI NOR**, 운영체제가 **UFS**에 있습니다.
"live USB 부팅"은 **U-Boot가 USB를 먼저 스캔해서 USB의 Armbian을 부팅하는 것**을 뜻합니다.

공장 상태에서는 공장 U-Boot가 USB를 스캔하지 않으므로, 부트 경로를 먼저 Armbian으로 바꿔야 합니다.
이를 어디까지 Armbian 방식으로 하느냐에 따라 경로가 3개입니다.

| 경로 | 시작 상태 | 쓰는 도구 | SPI NOR | UFS | 난이도 |
|---|---|---|---|---|---|
| **A** | 이미 Armbian U-Boot | `armbian-install` | 자동 | 자동 | ★☆☆ |
| **B** (권장) | 공장 펌웨어 | `fastboot` + `armbian-install` | fastboot로 기록 | `armbian-install` | ★★☆ |
| **C** | 공장 펌웨어 | 공장 U-Boot 자체(`flash_image usb`) | U-Boot가 기록 | U-Boot가 기록 | ★★☆ |

모두 TitanTools/Windows를 쓰지 않습니다. 경로 B는 **표준 fastboot 프로토콜**만 씁니다.

### 0.1 왜 공장 U-Boot는 USB로 바로 부팅하지 못하나

공장 U-Boot 바이너리(`u-boot.itb`)를 문자열로 확인하면:

```bash
strings -n 8 factory/u-boot.itb | grep -c 'distro_bootcmd'   # 0  → 표준 distro boot 없음
strings -n 8 factory/u-boot.itb | grep -c 'udisk boot'      # 0  → USB 디스크 자동 탐색 없음
strings -n 8 factory/u-boot.itb | grep -c 'flash_image'     # >0 → flash_image 명령은 있음
```

- 공장 U-Boot는 `boot_device` 변수에 따라 `nor_boot`/`ufs_boot`/`mmc_boot`/`udisk_boot`을 고르는 방식입니다.
  `udisk_boot` 명령은 env에 정의돼 있지만, **`boot_device=udisk`로 만들어 주는 코드
  (`CONFIG_BOOT_FROM_USB_DISK`)가 컴파일되어 있지 않습니다.**
  (Armbian 빌드도 동일하게 `CONFIG_BOOT_FROM_USB_DISK is not set`.)
- `fastboot`와 `flash_image` 명령은 컴파일되어 있습니다. → 경로 B/C가 가능합니다.

### 0.2 검증 상태 (중요)

| 경로 | Armbian 파일만 사용 | 우리 검증 범위 |
|---|---|---|
| **A** live USB → `armbian-install` | O | U-Boot/boot.scr/cmdline/NOR 일치/MPXY/팬/온도는 실기 검증. `armbian-install` end-to-end는 **미검증** |
| **B** fastboot → live USB → `armbian-install` | O | NOR 파일·레이아웃·버전은 검증. **물리 fastboot 모드 동작은 미검증**(사용자가 Booting 버튼으로 진행) |
| **C** 공장 U-Boot `flash_image usb` | O | 공장 U-Boot 바이너리/JSON 레이아웃은 확인. 부팅 메뉴 노출 여부는 보드 상태에 따라 다름 |

직접 확인하지 않은 부분은 문서에 "미검증"으로 표시했습니다.
실행 후 결과가 다르면 피드백을 주시면 문서를 갱신하겠습니다.

### 0.3 Armbian U-Boot는 USB를 먼저 스캔한다

Armbian U-Boot의 env(`u-boot-env-default.bin`):

```text
autoboot=... setenv devtype usb; usb start; sleep 2; run bootfunc;
          setenv devtype mmc; run bootfunc;
          setenv devtype nvme; pci enum; nvme scan; nvme part; run bootfunc;
          setenv devtype scsi; run bootfunc; ...
bootcmd=run autoboot;
bootfunc=... elif test -e ${devtype} ${devnum}:${distro_bootpart} /boot.scr; then
               load ${devtype} ${devnum}:${distro_bootpart} ${kernel_addr_r} /boot.scr; source ${kernel_addr_r}; ...
```

- `devtype=usb`, `devnum=0`, `distro_bootpart=1` → USB의 1번 파티션에 있는 `/boot.scr`를 찾습니다.
- U-Boot 시작 시 USB를 가장 먼저 스캔하므로, **USB가 꽂혀 있으면 UFS보다 USB가 우선**됩니다.
- `bootdelay=2`라 2초 안에 키를 누르면 U-Boot 프롬프트로 들어갈 수 있습니다.

---

## 1. 사전 준비물

| 준비물 | 설명 |
|---|---|
| Ubuntu/x86_64 빌드 호스트 | Armbian 빌드 (RISC-V 보드에서 빌드하지 않음) |
| AIBOX-K3 본체 + 전원 | |
| USB-A ↔ USB-C 케이블 1개 | **콘솔(Serial)** 용. 115200 8N1 |
| USB-A ↔ USB-C 케이블 1개 | **플래싱/데이터** 용 (보드의 USB-C DRD 포트) |
| USB 메모리 (8GB 이상) | live USB 용. 이미지 2.7GB + 여유 |
| `fastboot` (호스트 PC) | `sudo apt install fastboot` (Android platform-tools) |
| 시리얼 터미널 | `picocom -b 115200 /dev/ttyACM0` 또는 `screen /dev/ttyACM0 115200` |

포트 구분 (혼동 주의):

- **Console 포트**: 부팅 로그용. UART 브리지(CDC ACM)로 보이며 호스트에서 `/dev/ttyACM0`로 잡힙니다.
- **플래싱 포트 (USB-C DRD)**: fastboot/USB 저장장치용. Console 포트와 **다른** 포트입니다.

---

## 2. 산출물 만들기

### 2.1 빌드

```bash
cd ~/code/armbian/build
sudo ./compile.sh requirements        # 처음 한 번만

./compile.sh build BOARD=firefly-aibox-k3 BRANCH=legacy \
  BUILD_DESKTOP=no BUILD_MINIMAL=no EXPERT=yes \
  KERNEL_CONFIGURE=no RELEASE=resolute
```

### 2.2 필요한 파일 모으기

Armbian U-Boot deb 안에 SPI NOR 기록용 파일이 모두 들어 있습니다.

```bash
cd ~/code/armbian/build
DEB=$(ls -t output/debs/linux-u-boot-firefly-aibox-k3-legacy_*.deb | head -1)
rm -rf /tmp/aibox-nor && mkdir -p /tmp/aibox-nor
dpkg-deb -x "$DEB" /tmp/uboot-pkg
U=/tmp/uboot-pkg/usr/lib/linux-u-boot-legacy-firefly-aibox-k3

cp "$U/bootinfo_spinor.bin"     /tmp/aibox-nor/
cp "$U/FSBL.bin"                /tmp/aibox-nor/
cp "$U/u-boot-env-default.bin"  /tmp/aibox-nor/
cp "$U/esos.itb"                /tmp/aibox-nor/
cp "$U/fw_dynamic.itb"          /tmp/aibox-nor/
cp "$U/u-boot.itb"              /tmp/aibox-nor/
cp "$U/partition_4M.json"       /tmp/aibox-nor/
cp "$U/spi-flash-howto.txt"     /tmp/aibox-nor/
ls -l /tmp/aibox-nor
```

기대:

```text
bootinfo_spinor.bin        80
FSBL.bin               453,984
u-boot-env-default.bin  16,384
esos.itb               506,499
fw_dynamic.itb         271,311
u-boot.itb           2,683,538
partition_4M.json          968
spi-flash-howto.txt        683
```

`partition_4M.json`은 NOR 레이아웃과 각 파티션에 쓸 파일을 지정합니다.

```json
{ "name": "bootinfo", "offset": "0",    "size": "128K", "image": "bootinfo_spinor.bin" },
{ "name": "fsbl",     "offset": "128K", "size": "512K", "image": "FSBL.bin" },
{ "name": "env",      "offset": "640K", "size": "64K",  "image": "u-boot-env-default.bin" },
{ "name": "esos",     "offset": "704K", "size": "1M",   "image": "esos.itb" },
{ "name": "opensbi",  "offset": "1728K","size": "384K", "image": "fw_dynamic.itb" },
{ "name": "uboot",    "offset": "2112K","size": "3M",   "image": "u-boot.itb" }
```

`image` 경로는 **JSON 파일과 같은 디렉터리 기준**입니다. 위 `cp`처럼 한 폴더에 모으세요.

> 참고: JSON의 `uboot` 크기("3M")는 NOR 끝까지 쓰는 Linux `mtdparts`(-@2112K)와 다릅니다.
> `uboot`이 마지막 파티션이라 실제로는 영향이 없습니다. 파일(2,683,538 B)만 기록됩니다.

### 2.3 UFS에 설치할 이미지

```bash
IMG=$(ls -t output/images/*.img | head -1)
ls -lh "$IMG"
cat "${IMG}.sha"
```

---

## 3. live USB 만들기

Armbian 이미지 전체를 USB에 그대로 씁니다. 부트 파티션(1번)과 rootfs(2번)가 모두 들어 있어야 합니다.

```bash
lsblk -o NAME,SIZE,TYPE,MODEL,TRAN        # USB 장치 식별 (예: /dev/sdb)
sudo dd if="$IMG" of=/dev/sdX bs=4M status=progress conv=fsync
sync
```

확인:

```bash
sudo blkid /dev/sdX*            # 파티션 UUID/라벨
sudo parted /dev/sdX unit MiB print   # 파티션 번호와 이름
```

Armbian 이미지의 GPT 파티션 이름은 `armbi_boot` / `armbi_root`입니다.

```text
/dev/sdX1  256MiB  armbi_boot   (FAT32, boot.scr/Image/uInitrd/dtb)
/dev/sdX2  ~2.5G   armbi_root   (ext4,  루트파일시스템)
```

### 3.1 라벨 이름을 바꿔야 하는가

- **Armbian U-Boot로 부팅**: 필요 없습니다. `bootfunc`는 라벨이 아니라 **파티션 번호**
  (`distro_bootpart=1`)와 파일명(`/boot.scr`)으로 찾습니다.
- **공장 U-Boot로 부팅**하려면 라벨이 `bootfs` / `rootfs`여야 합니다
  (공장 env의 `part number usb 0 bootfs`, `udisk_add_args`가 라벨을 찾습니다).
  경로 C에서 공장 U-Boot를 쓸 때만 필요합니다.

```bash
# 필요할 때만 (공장 U-Boot용)
sudo parted /dev/sdX name 1 bootfs
sudo parted /dev/sdX name 2 rootfs
```

---

## 4. 경로 A: Armbian U-Boot가 이미 있는 보드

이미 SPI NOR에 Armbian U-Boot가 있고 UFS에 Armbian이 설치된 상태라면 가장 간단합니다.

1. AIBOX 전원 **끄기**
2. live USB를 보드의 USB 포트에 연결
3. 전원 켜기
4. 시리얼 콘솔에서 부팅 확인

```text
U-Boot 2022.10_armbian-2022.10-Se471-P78cf-Hde31-V0a62-B56a7-R448a
Loading K3 environment
Loading Armbian environment (usb 0:1)
Booting Armbian (usb 0:1)
Starting kernel ...
```

5. 부팅한 시스템에서 UFS로 설치 → [7장](#7-armbian-install-실행)

---

## 5. 경로 B: 공장 상태에서 시작 (fastboot)

**권장 경로**입니다. 표준 fastboot 프로토콜로 SPI NOR에 Armbian 펌웨어 세트를 기록하고,
그다음 Armbian U-Boot가 USB를 스캔하므로 live USB → `armbian-install`로 마무리합니다.

### 5.1 왜 fastboot를 쓸 수 있나

- 공장 SPL(`factory/FSBL.bin`)에 fastboot가 내장되어 있습니다.
- Armbian U-Boot 빌드에도 fastboot가 있습니다.

```text
CONFIG_SPL_FASTBOOT=y
CONFIG_CMD_FASTBOOT=y
CONFIG_FASTBOOT=y
CONFIG_FASTBOOT_FLASH=y
CONFIG_FASTBOOT_MULTI_FLASH_OPTION=y
CONFIG_FASTBOOT_MULTI_FLASH_OPTION_MTD=y
CONFIG_FASTBOOT_GPT_NAME="gpt"
```

`CONFIG_FASTBOOT_MULTI_FLASH_OPTION_MTD`가 `fastboot flash mtd <json>`을 제공합니다.

### 5.2 fastboot 모드 진입

1. AIBOX 전원 **분리**
2. **플래싱 포트(USB-C DRD)**를 호스트에 연결
3. **Boot 버튼을 누른 상태로** 전원 연결
4. 호스트에서:

```bash
fastboot devices -l
```

```text
????????????           DFU download
 usb:3-3
```

### 5.3 Armbian 펌웨어를 SPI NOR에 기록 (벤더 howto 그대로)

`/tmp/aibox-nor` 디렉터리에서 실행합니다(6개 파일 + `partition_4M.json`이 함께 있어야 함).

```bash
cd /tmp/aibox-nor
ls -l          # 6개 파일 + partition_4M.json 확인

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

이 시퀀스는 U-Boot 패키지의 `spi-flash-howto.txt`에 들어 있는 것과 같은 flow입니다.

```text
# Connect OTG to PC and put the unit into fastboot mode
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

각 명령이 하는 일(U-Boot 소스 기준, `drivers/fastboot/fb_command.c`, `fb_mtd.c`, `fb_spacemit.c`):

1. `fastboot stage FSBL.bin` → Armbian SPL을 RAM에 올림 → `continue`로 SPL 실행
2. SPL이 fastboot 모드로 진입 → `fastboot oem speed:super-speed`로 고속 모드
3. `fastboot stage u-boot.itb` → Armbian U-Boot를 RAM에 올림 → `continue`로 U-Boot 실행
4. `fastboot flash mtd partition_4M.json`
   - 호스트가 JSON을 디바이스 버퍼로 다운로드한 뒤 `flash:mtd`를 보냅니다.
   - 디바이스는 버퍼의 JSON을 파싱해 **NOR 파티션 테이블(mtdids/mtdparts)** 을 설정합니다
     (`_parse_flash_config()` → `_write_mtd_partition()`).
   - 즉 이 명령은 "이 레이아웃으로 NOR을 쓰겠다"를 알려주는 단계입니다.
5. `fastboot reboot` → 재부팅, SPI NOR은 이제 Armbian 세트

> 실제 파일 기록은 호스트 도구가 JSON을 읽어 파티션별로 issuing합니다.
> TitanTools는 이걸 `multi_flash`(fastboot.yaml의 `relate_partition`)로 처리합니다.
> 순수 AOSP `fastboot` 클라이언트에서는 5.4의 변형을 쓰세요.

### 5.4 AOSP fastboot으로 파티션별 기록 (변형)

`partition_4M.json`의 파티션 이름/오프셋은 NOR 파티션 이름(`bootinfo`, `fsbl`, `env`,
`esos`, `opensbi`, `uboot`)과 같습니다. 따라서 JSON을 먼저 먹이고, 파일을 파티션별로 보낼 수 있습니다.

```bash
cd /tmp/aibox-nor

# 1) NOR 레이아웃을 알려준다
fastboot flash mtd partition_4M.json

# 2) 파티션별로 파일 기록
fastboot flash bootinfo bootinfo_spinor.bin
fastboot flash fsbl     FSBL.bin
fastboot flash env      u-boot-env-default.bin
fastboot flash esos     esos.itb
fastboot flash opensbi  fw_dynamic.itb
fastboot flash uboot    u-boot.itb

fastboot reboot
```

> 주의: NOR 모드에서 U-Boot fastboot은 "hidden bootloader partition" 이름을 건너뛰도록 되어 있습니다
> (`is_bootloader_hidden_partition()`). 따라서 위 변형이 정상 동작하는지는 **물리적으로 fastboot 모드에서
> 직접 확인해야 합니다.** 건너뛰면 아무 것도 기록되지 않은 채 재부팅됩니다.
> 실패하면 TitanTools의 `multi_flash`를 쓰거나, 6장(공장 U-Boot의 `flash_image`)으로 우회하세요.

### 5.5 기록 결과 확인

부팅 후 시리얼에서:

```text
U-Boot SPL 2022.10_armbian-2022.10-Se471-P78cf-Hde31-V0a62-B56a7-R448a
U-Boot 2022.10_armbian-2022.10-Se471-P78cf-Hde31-V0a62-B56a7-R448a
Loading K3 environment
```

UFS가 아직 공장 상태이면 여기서 `nor_boot`/`ufs_boot`가 실패할 수 있습니다(공장 UFS에는 `env_k3.txt`가 없음).
**그 상태가 정상입니다.** USB를 꽂고 재부팅하면 Armbian live USB로 부팅됩니다.

```bash
# 부팅 가능한 상태를 만들려면
# AIBOX 전원 끄기 → USB 연결 → 전원 켜기
```

NOR에 기록된 내용은 **Armbian에서 직접 확인**할 수 있습니다(live USB 부팅 후).

```bash
U=/usr/lib/linux-u-boot-legacy-firefly-aibox-k3
for pair in "bootinfo_spinor.bin mtd1" "FSBL.bin mtd2" "u-boot-env-default.bin mtd3" \
            "esos.itb mtd4" "fw_dynamic.itb mtd5" "u-boot.itb mtd6"; do
  set -- $pair
  size=$(stat -c%s "$U/$1")
  cmp -n "$size" "$U/$1" "/dev/$2" && echo "$2 MATCH" || echo "$2 MISMATCH"
done
```

### 5.6 fastboot이 실패할 때

| 증상 | 확인할 것 |
|---|---|
| `fastboot devices`가 아무것도 안 뜸 | Boot 버튼을 누른 채 전원 연결했는지, **플래싱 포트**를 썼는지(Console 포트가 아님) |
| `fastboot continue` 후 멈춤 | `sleep 10`을 지키는지. 안 되면 `fastboot devices`로 살아 있는지 확인 |
| `fastboot flash mtd partition_4M.json`이 파일을 못 찾음 | JSON과 6개 파일이 **같은 디렉터리**에 있는지, 실행 위치가 그 폴더인지 |
| `fastboot oem speed:super-speed`가 실패 | 무시해도 진행 가능. 이미 고속으로 붙어 있으면 발생하지 않습니다 |
| 기록 후에도 공장 U-Boot로 부팅됨 | NOR 기록이 안 된 것. 5.4 참고 |

### 5.7 fastboot 없이 하고 싶은 경우

TitanTools 없이 **U-Boot 안에서** 기록하는 방법이 있습니다 → [6장](#6-경로-c-공장-u-boot의-recovery-from-usb-사용).

---

## 6. 경로 C: 공장 U-Boot의 "recovery from usb" 사용

공장 U-Boot에는 `flash_image` 명령이 컴파일되어 있고, env에 다음 항목이 있습니다.

```text
bootmenu_8="recovery from usb"=run flash_from_usb
flash_from_usb=echo "recovery from usb...... "; flash_image usb;
```

`flash_image usb`는 USB에 있는 `partition_flash.json`을 읽고 그에 적힌 파일들을
디스크/GPT에 기록하는 Firefly 자체 기능입니다.

### 6.1 필요한 USB sticks

USB에는 GPT 파티션이 **이름 `bootfs`**로 있어야 하고, 그 안에 다음 파일이 있어야 합니다.

```text
bootfs/  (FAT32, GPT 파티션 이름 = "bootfs")
├── partition_flash.json
├── env.bin                  (Armbian U-Boot env)
├── bootinfo_spinor.bin
├── FSBL.bin                 (Armbian SPL)
├── u-boot.itb               (Armbian U-Boot)
├── esos.itb                 (Armbian ESOS)
├── fw_dynamic.itb           (Armbian OpenSBI)
├── bootfs.img               (Armbian bootfs)
└── rootfs.ext4              (Armbian rootfs)
```

`partition_flash.json`은 공장 패키지(`partition_flash.json`)를 쓰면 됩니다.
다만 그 JSON은 **UFS GPT 레이아웃**을 가리키고 파일 경로가 `factory/...` 형태입니다.
Armbian 파일로 바꾸려면 사본을 만들고 `image` 경로를 Armbian 파일명으로 바꿉니다.

```json
{ "name": "env",     "offset": "640K", "size": "64K",  "image": "env.bin" },
{ "name": "bootfs",  "offset": "12M",  "size": "-",    "image": ["env.bin", "bootinfo_spinor.bin", "FSBL.bin", "u-boot.itb", "esos.itb", "fw_dynamic.itb", "bootfs.img", "rootfs.ext4"] }
```

> 이 경로는 공장 U-Boot의 부팅 메뉴에 "recovery from usb"가 보이는지에 따라 사용 가능합니다.
> 화면에 메뉴가 안 보이면 경로 B(fastboot)를 쓰세요.

### 6.2 주의

- 이 경로는 **UFS(GPT)** 를 기록하는 쪽입니다. SPI NOR은 공장 U-Boot로 남습니다.
  그 상태도 Armbian UFS는 부팅되지만(`env_k3.txt` 덕분), 최종 상태는 경로 B가 더 깔끔합니다.
- 공장 U-Boot는 라벨 `bootfs` / `rootfs`를 요구합니다 → [3.1](#31-라벨-이름을-바꿔야-하는가)

---

## 7. armbian-install 실행

live USB로 부팅한 Armbian에서 실행합니다. 이 단계는 **UFS에 rootfs를 복사하고,
SPI NOR에 U-Boot를 기록**합니다.

### 7.1 사전 확인

```bash
# 1) 지금 USB로 부팅했는가
findmnt -no SOURCE /
lsblk -o NAME,SIZE,TYPE,MODEL,TRAN,MOUNTPOINTS
```

```text
NAME   SIZE TYPE  MODEL            TRAN      MOUNTPOINTS
sda    119G disk  UFS
├─sda1  256M part                   vfat
└─sda2   2.5G part                   ext4
sdb    14.8G disk  USB               usb
├─sdb1  256M part                   vfat      /boot
└─sdb2   2.5G part                   ext4      /
```

- 루트(`/`)가 **USB** 쪽이어야 합니다 (UFS가 아니라).
- 설치 대상은 **UFS**(`/dev/sda`)입니다.

```bash
# 2) SPI NOR 파티션이 보이는가
cat /proc/mtd
```

```text
mtd0: 00800000 ... "d420c000.spi"
mtd1: 00020000 ... "bootinfo"
mtd2: 00080000 ... "fsbl"
mtd3: 00010000 ... "env"
mtd4: 00100000 ... "esos"
mtd5: 00060000 ... "opensbi"
mtd6: 005f0000 ... "uboot"
```

`mtdparts`가 cmdline에 없으면 `/proc/mtd`가 비어 있습니다.
이미지의 `armbianEnv.txt`에 `extraargs=... mtdparts=...`가 들어 있으므로 정상 부팅이면 보입니다.

```bash
# 3) U-Boot 패키지와 설치 함수가 있는가
dpkg -l | grep linux-u-boot-firefly-aibox-k3-legacy
grep -n write_uboot_platform_mtd /usr/lib/u-boot/platform_install.sh
```

### 7.2 백업 (선택이지만 권장)

UFS는 `armbian-install` 과정에서 **지워집니다**. 기존 데이터가 필요하면 먼저 백업하세요.

```bash
# UFS 파티션만 백업 (이미지 파일로)
sudo dd if=/dev/sda1 of=/tmp/bootfs-backup.img bs=4M status=progress
sudo dd if=/dev/sda2 of=/tmp/rootfs-backup.img bs=4M status=progress
```

### 7.3 대화형 설치

```bash
sudo armbian-install
```

선택 항목:

| 항목 | 값 |
|---|---|
| target | UFS (`/dev/sda`) |
| boot | `SPI/MTD` (또는 `mtd`) |
| filesystem | `ext4` |

### 7.4 비대화형 설치

```bash
sudo armbian-install --target /dev/sda --boot mtd --fs ext4 --yes
```

### 7.5 내부 동작

```text
1) 실행 중인 rootfs(=USB의 Armbian)를 target으로 복사
   - bootfs/rootfs 파티션 생성 후 rsync
   - /boot/armbianEnv.txt의 rootdev를 target의 UUID로 다시 기록
2) --boot mtd
   - /proc/mtd에서 대상 목록 생성
   - /usr/lib/u-boot/platform_install.sh의 write_uboot_platform_mtd() 호출
   - NOR 6개 파티션에 bootinfo/FSBL/env/esos/opensbi/u-boot 기록
3) 로그: /var/log/armbian-install.log
```

`write_uboot_platform_mtd()`는 **이름으로 파티션을 찾아 NOR 전체를 기록**합니다.

```bash
cat /var/log/armbian-install.log
cat /tmp/spinor.log        # 구현에 따라
```

### 7.6 주의

- **대상 디스크를 꼭 확인하세요.** `/dev/sdX`를 잘못 쓰면 USB가 지워집니다.
- `armbian-install`은 **실행 중인 rootfs가 있는 디스크를 target으로 삼지 않습니다.**
  그래서 live USB가 필수입니다.
- 설치 중 전원/케이블을 뽑지 마세요.
- bootloader를 NOR에 쓰기 때문에 `mtd` 모드가 아니면 SPI NOR은 factory 그대로 남습니다.

---

## 8. 재부팅 후 검증

1. **USB를 뽑습니다.** (UFS로 부팅해야 합니다)
2. 재부팅
3. 시리얼 콘솔:

```text
U-Boot SPL 2022.10_armbian-2022.10-Se471-P78cf-Hde31-V0a62-B56a7-R448a
U-Boot 2022.10_armbian-2022.10-Se471-P78cf-Hde31-V0a62-B56a7-R448a
Loading K3 environment
Loading Armbian environment (scsi 0:1)
Booting Armbian (scsi 0:1)
Starting kernel ...
```

`scsi 0:1`이 보이면 UFS에서 부팅한 것입니다(USB가 아니라).

4. 시스템 확인:

```bash
cat /proc/cmdline                 # mtdparts 포함, root=PARTUUID(UFS)
cat /proc/mtd                     # NOR 6개 파티션
dmesg | grep -i mpxy              # mailbox registered with 5 channels
cat /sys/class/thermal/thermal_zone0/temp
cat /sys/class/hwmon/hwmon0/fan1_input
systemctl --failed                 # 0 loaded units
```

5. NOR 기록 일치 확인:

```bash
U=/usr/lib/linux-u-boot-legacy-firefly-aibox-k3
for pair in "bootinfo_spinor.bin mtd1" "FSBL.bin mtd2" "u-boot-env-default.bin mtd3" \
            "esos.itb mtd4" "fw_dynamic.itb mtd5" "u-boot.itb mtd6"; do
  set -- $pair
  size=$(stat -c%s "$U/$1")
  cmp -n "$size" "$U/$1" "/dev/$2" && echo "$2 MATCH" || echo "$2 MISMATCH"
done
```

6. `armbian-config`로 후속 설정(사용자 생성, 네트워크 등)을 진행합니다.

---

## 9. 복구

### 9.1 부팅이 안 될 때 (NOR/UFS 문제)

fastboot로 NOR을 다시 기록하면 됩니다. UFS는 지워졌을 수 있으니 USB로 부팅해 다시 설치합니다.

```bash
# 1) 전원 분리 → 플래싱 포트 연결 → Boot 버튼 누른 채 전원
fastboot devices -l
# 2) NOR 재기록
cd /tmp/aibox-nor
fastboot stage FSBL.bin && fastboot continue && sleep 10
fastboot oem speed:super-speed
fastboot stage u-boot.itb && fastboot continue && sleep 3
fastboot flash mtd partition_4M.json
fastboot reboot
# 3) USB 꽂고 재부팅 → live USB → armbian-install 재실행
```

### 9.2 공장 펌웨어로 되돌리기 (TitanTools 없이)

fastboot에 **공장** 파일을 넣으면 됩니다. 공장 패키지에서 가져온 파일이 필요합니다.

```text
factory/FSBL.bin          (공장 SPL)
factory/bootinfo_spinor.bin
factory/u-boot.itb        (공장 U-Boot)
env.bin                   (공장 env)
esos.itb, fw_dynamic.itb  (공장 펌웨어)
```

주의: **env.bin은 공장 env여야 합니다.** Armbian env를 넣으면 공장 U-Boot 대신
Armbian U-Boot 환경으로 부팅하므로 "공장으로 복구"가 되지 않습니다.

### 9.3 공장 UFS 파티션 구조 (참고)

공장은 UFS GPT에도 부트로더 파티션을 둡니다(`partition_universal.json`).

| 이름 | offset | 크기 | 내용 |
|---|---|---:|---|
| `env` | 640K | 64K | U-Boot env |
| `bootinfo` | 1M | 128K | BootROM용 정보 |
| `fsbl` | 1536K | 512K | U-Boot SPL |
| `esos` | 4M | 3M | ESOS |
| `opensbi` | 7M | 1M | OpenSBI |
| `uboot` | 8M | 4M | U-Boot proper |
| `bootfs` | 12M | 256M | FAT bootfs |
| `rootfs` | 268M | 나머지 | ext4 rootfs |

Armbian의 UFS 파티션은 `bootfs`(256M FAT) + `rootfs`(나머지 ext4) **두 개뿐**이고,
부트로더는 SPI NOR에만 있습니다. 이 차이를 confuse하지 않도록 주의하세요.

---

## 10. 부록

### 10.1 한눈에 보는 부팅 경로

```text
[공장 상태]
  BootROM -> bootinfo(NOR) -> FSBL -> OpenSBI -> U-Boot(공장)
                                               |
      경로 B: fastboot로 NOR에 Armbian 세트 기록
      경로 C: 공장 U-Boot의 flash_image usb
                                               |
                                               v
[Armbian 상태]
  BootROM -> bootinfo(NOR) -> FSBL -> OpenSBI -> U-Boot(Armbian)
                                               |
      USB가 꽂혀 있으면: USB의 /boot.scr -> live Armbian
      USB가 없으면:      UFS의 /boot.scr -> 설치된 Armbian
```

### 10.2 자주 쓰는 명령

```bash
# fastboot
fastboot devices -l
fastboot getvar all
fastboot stage <file> && fastboot continue
fastboot flash mtd partition_4M.json
fastboot reboot

# U-Boot 콘솔에서 (bootdelay=2 동안 아무 키)
usb start
usb list
part list usb 0
sf probe; sf read <addr> 0 <len>     # NOR 읽기
printenv boot_device
run autoboot
```

### 10.3 주의 사항

| 항목 | 내용 |
|---|---|
| `erasespi` | NOR의 4MiB만 지웁니다. 전체 지우기가 아닙니다. 복구 계획 없이 쓰지 마세요 |
| `fastboot flash mtd` | JSON과 이미지 파일이 **같은 디렉터리**에 있어야 합니다 |
| USB 파티션 라벨 | Armbian U-Boot는 무관, 공장 U-Boot는 `bootfs`/`rootfs` 필요 |
| `armbian-install` | 실행 중인 rootfs의 복사본을 target에 씁니다. live USB가 필수 |
| NOR 백업 | `write_uboot_platform_mtd()`는 자동 백업하지 않습니다. 수동으로 하세요 |
| U-Boot 버전 문자열 | 패치/ATF 디렉터리/blob을 바꾸면 deb 버전 문자열이 바뀝니다 → 이미지·패키지·문서 SHA도 다시 |

### 10.4 참고 자료

- Armbian build framework: <https://github.com/armbian/build>
- Armbian installer 문서: <https://docs.armbian.com/User_Basic_Manual>
- Firefly AIBOX-K3 wiki: <https://wiki.t-firefly.com/en/AIBOX-K3/>
- Firefly AIBOX-K3 UART debug: <https://community.t-firefly.com/en/docs/products/computers/AIBOX-K3/debug>
- Firefly firmware upgrade: <https://wiki.t-firefly.com/en/AIBOX-K3/upgrade_firmware.html>
- U-Boot fastboot: <https://docs.u-boot.org/en/latest/usage/fastboot.html>
