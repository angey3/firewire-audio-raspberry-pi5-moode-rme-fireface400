# Raspberry Pi 5にFireWireを搭載してRME Fireface 400をmoOde Audio Playerで動かす

Raspberry Pi 5にMini PCIe FireWireカードを搭載し、RME Fireface 400をmoOde Audio Playerで動作させるための手順です。

本手順を適用することで、Raspberry Pi 5上のmoOde Audio Playerから全サンプリングレート（44.1kHz〜192kHz）でFireface 400を使用した安定した再生が可能になります。

なお、本リポジトリは個人的な検証に基づくものです。動作を保証するものではなく、サポートは提供できません。自己責任でご使用ください。

---

## 対象機器

- **オーディオインターフェース**：RME Fireface 400
- **ストリーマー**：Raspberry Pi 5B
- **OS / プレーヤーソフト**：moOde Audio Player 10.1.2（Debian Trixie ベース）
- **カーネル**：Linux 6.12.75（`6.12.75+rpt-rpi-2712`）

---

## 必要なハードウェア

### Raspberry Pi 5

本手順はRaspberry Pi 5専用です。Pi 4以前では動作しません。

### PCIe to Mini PCIe 変換HAT

- **WaveshareのPCIe to Mini PCIe HAT+**（型番：31534）
- スイッチサイエンス、共立エレショップなどで入手可能（約1,650円）
- Raspberry Pi 5専用設計

### Mini PCIe FireWireカード

- **StarTech MPEX1394B3**（TI XIO2213A/BチップまたはXIO2221チップ搭載）

> **重要：製造年による違いについて**
> StarTech MPEX1394B3は製造時期によって基板が異なります。
> - **2016年製造品（基板Rev 643ME）**：著者環境ではFireface 400を認識しませんでした。相性なのか、製品自体の良否なのかを確認中です。
> - **2020年製造品（基板Rev 2213ME V1.1）**：正常に動作を確認しています。
>
> 購入時は製造年や基板リビジョンの確認をお勧めします。なお、開封品や返品品には注意してください。

### FireWireケーブル

以下のいずれかで動作を確認しています：
- FW400（6ピン）→ FW400（6ピン）ケーブル
- FW800（9ピン）→ FW400（6ピン）変換ケーブル

### RME Fireface 400

外部電源で動作させることを推奨します。StarTech MPEX1394B3にはFireWireバスパワー供給用の外部電源ケーブルが付属していますが、Fireface 400は外部電源があるため使用不要です。

---

## 動作確認環境

- **ストリーマー**：Raspberry Pi 5B 4GB
- **ケース**：GeeekPi製メタルケース（Raspberry Pi 5用、アクティブクーラー付き、PCIe周辺機器ボード対応）
- **HAT**：WaveshareのPCIe to Mini PCIe HAT+（31534）
- **FireWireカード**：StarTech MPEX1394B3（2020年製、基板Rev 2213ME V1.1）
- **電源**：USB-C PD対応アダプター 27W
- **NAS**：アイ・オー・データ Soundgenic HDL-RA2HF
- **ネットワーク**：有線LAN
- **コントロール**：fidata Music App（iOS）/ LUMIN（iOS）
- **接続プロトコル**：OpenHome

---

## 問題の背景

Raspberry Pi 5はPCIeスロットを持っていますが、FireWireコントローラーは搭載されていません。また、標準のmoOdeカーネルはFireWireオーディオモジュールを含んでいません。

さらに、Pi5のPCIeはデフォルトで32ビットDMAが無効になっており、これを有効にしないとFireWireコントローラーが正常に動作しません。

本手順ではこれらの問題を解決し、Pi5上でFireface 400をALSAオーディオデバイスとして動作させます。

---

## インストール手順

### 前提条件
- moOde Audio Playerがインストール済みで起動していること
- SSHで接続できること
- インターネットに接続されていること
- WaveshareのHATとStarTech FireWireカードが装着済みであること

### 1. システムの更新

```bash
sudo apt update && sudo apt upgrade -y
sudo reboot
```

### 2. カーネルバージョンの確認

再接続後に確認します：

```bash
uname -r
```

`6.12.75+rpt-rpi-2712`と表示されれば正常です。

### 3. config.txtの設定

```bash
sudo nano /boot/firmware/config.txt
```

以下を追加します：

```
dtparam=pciex1
dtoverlay=pcie-32bit-dma
dtparam=pciex1_gen=2
```

### 4. cmdline.txtの設定

```bash
sudo nano /boot/firmware/cmdline.txt
```

末尾に以下を追加します（スペースを入れて）：

```
pcie_aspm=off
```

### 5. 設定を反映して再起動

```bash
sudo reboot
```

### 6. FireWireカードの認識確認

再起動後に確認します：

```bash
lspci
```

以下のように表示されれば正常です：

```
0001:01:00.0 PCI bridge: Texas Instruments XIO2213A/B/XIO2221 PCI Express to PCI Bridge
0001:02:00.0 FireWire (IEEE 1394): Texas Instruments XIO2213A/B/XIO2221 IEEE-1394b OHCI Controller
```

### 7. ビルドツールとgitのインストール

```bash
sudo apt install -y git build-essential bc bison flex libssl-dev libelf-dev
```

### 8. カーネルソースの取得

```bash
cd ~
git clone --depth 1 --branch rpi-6.12.y --no-checkout https://github.com/raspberrypi/linux.git rpi-linux
cd rpi-linux
git sparse-checkout init
git sparse-checkout set drivers/firewire sound/firewire
git checkout
```

### 9. ohci.cの修正（1箇所目）

TI XIO2213チップ専用のquirksエントリを追加します：

```bash
grep -n "PCI_DEVICE_ID_TI_TSB12LV22" ~/rpi-linux/drivers/firewire/ohci.c
```

表示された行番号の直前に以下を追加します：

```bash
nano +XXX ~/rpi-linux/drivers/firewire/ohci.c
```

以下の行の直前に：
```c
{PCI_VENDOR_ID_TI, PCI_DEVICE_ID_TI_TSB12LV22, PCI_ANY_ID,
```

以下を追加してください：
```c
{PCI_VENDOR_ID_TI, 0x823f, PCI_ANY_ID,
QUIRK_RESET_PACKET | QUIRK_TI_SLLZ059},
```

### 10. ohci.cの修正（2箇所目）

DMAマスクの設定を追加します：

```bash
grep -n "pci_set_master" ~/rpi-linux/drivers/firewire/ohci.c
```

表示された行番号の直後に以下を追加します：

```c
err = dma_set_mask_and_coherent(&dev->dev, DMA_BIT_MASK(64));
if (err) {
    err = dma_set_mask_and_coherent(&dev->dev, DMA_BIT_MASK(32));
    if (err) {
        ohci_err(ohci, "failed to set DMA mask\n");
        return err;
    }
}
```

### 11. FireWireドライバーのビルド

```bash
cd ~/rpi-linux
make -C /usr/src/linux-headers-6.12.75+rpt-rpi-2712 M=$(pwd)/drivers/firewire CONFIG_FIREWIRE=m CONFIG_FIREWIRE_OHCI=m modules 2>&1 | tail -10
```

### 12. snd-firefaceのビルド

```bash
make -C /usr/src/linux-headers-6.12.75+rpt-rpi-2712 M=$(pwd)/sound/firewire CONFIG_SND_FIREWIRE_LIB=m CONFIG_SND_FIREFACE=m KBUILD_EXTRA_SYMBOLS=$(pwd)/drivers/firewire/Module.symvers modules 2>&1 | tail -10
```

### 13. モジュールのインストール

```bash
sudo mkdir -p /lib/modules/$(uname -r)/kernel/drivers/firewire
sudo cp drivers/firewire/firewire-core.ko /lib/modules/$(uname -r)/kernel/drivers/firewire/
sudo cp drivers/firewire/firewire-ohci.ko /lib/modules/$(uname -r)/kernel/drivers/firewire/
sudo mkdir -p /lib/modules/$(uname -r)/kernel/sound/firewire/fireface
sudo cp sound/firewire/snd-firewire-lib.ko /lib/modules/$(uname -r)/kernel/sound/firewire/
sudo cp sound/firewire/fireface/snd-fireface.ko /lib/modules/$(uname -r)/kernel/sound/firewire/fireface/
sudo depmod -a
```

### 14. シャットダウン時のモジュールアンロード設定

> **注意：** この設定を行わないとシャットダウン・再起動時にシステムが固まる場合があります。モジュールインストール後、最初の再起動前に必ず設定してください。

FireWireモジュールがシャットダウン時にハングする問題を防ぐため、シャットダウン前にモジュールをアンロードするサービスを作成します：

```bash
sudo bash -c 'cat > /etc/systemd/system/firewire-cleanup.service << EOF
[Unit]
Description=Unload FireWire modules before shutdown
DefaultDependencies=no
Before=shutdown.target reboot.target halt.target
Requires=shutdown.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStop=/sbin/modprobe -r snd_fireface
ExecStop=/sbin/modprobe -r snd_firewire_lib
ExecStop=/sbin/modprobe -r firewire_ohci
ExecStop=/sbin/modprobe -r firewire_core

[Install]
WantedBy=multi-user.target
EOF'
sudo systemctl enable firewire-cleanup.service
```

またシャットダウンのタイムアウトを設定します：

```bash
sudo nano /etc/systemd/system.conf
```

以下の行のコメントを外して変更します：

```
DefaultTimeoutStopSec=15s
```

```bash
sudo systemctl daemon-reload
```

### 15. モジュールの自動ロード設定

起動時にFireWireコアモジュールを自動ロードします：

```bash
sudo bash -c 'cat > /etc/modules-load.d/firewire-audio.conf << EOF
firewire_ohci
EOF'
```

Fireface 400接続時に自動でオーディオモジュールをロードするudevルールを設定します：

```bash
sudo bash -c 'cat > /etc/udev/rules.d/91-snd-fireface.rules << EOF
SUBSYSTEM=="firewire", ACTION=="add", ATTR{units}=="*0x00a35:0x000001*", RUN+="/sbin/modprobe snd_firewire_lib", RUN+="/sbin/modprobe snd_fireface"
EOF'
sudo udevadm control --reload-rules
```

### 16. 再起動

```bash
sudo reboot
```

再起動後にFireWireカードとFireface 400が認識されているか確認します：

```bash
lspci | grep FireWire
aplay -l | grep Fireface
```

---

## moOde設定

### 17. Output deviceの設定

moOdeのWeb UIで **Configure** → **Audio** → **Output device** で「Fireface400」を選択して **Save** をクリックします。

### 18. ALSAデバイス設定の変更

moOdeのデフォルトALSAデバイス設定を変更します：

```bash
sudo nano /etc/alsa/conf.d/_audioout.conf
```

`type copy`を`type plug`に変更します：

```
pcm._audioout {
    type plug
    slave.pcm "hw:Fireface400,0"
}
```

```bash
sudo systemctl restart mpd
```

> **注意：** moOdeのWeb UIでOutput deviceを変更すると`_audioout.conf`が上書きされます。その場合は再度この設定を行ってください。

---

## 動作確認

Fireface 400が認識されているか確認します：

```bash
aplay -l | grep Fireface
```

再生中にサンプルレートを確認します（カード番号はaplay -lで確認してください）：

```bash
cat /proc/asound/cardX/pcm0p/sub0/hw_params
```

クロック設定を確認します：

```bash
cat /proc/asound/cardX/firewire/status
```

---

## 動作確認結果

以下のサンプリングレートでネイティブ出力（変換なし）を確認しています：

| サンプリングレート | 出力フォーマット | 結果 |
|------------|------------|------|
| 44.1kHz | S32_LE 44100Hz | ✅ 正常 |
| 48kHz | S32_LE 48000Hz | ✅ 正常 |
| 88.2kHz | S32_LE 88200Hz | ✅ 正常 |
| 96kHz | S32_LE 96000Hz | ✅ 正常 |
| 176.4kHz | S32_LE 176400Hz | ✅ 正常 |
| 192kHz | S32_LE 192000Hz | ✅ 正常 |
| DSD | S32_LE 192000Hz（PCM変換） | ✅ 正常 |

**FireWireのジッター対策について：**
FireWire（IEEE 1394）はアイソクロナス転送を採用しており、受信側（Fireface 400）のクロックを基準にデータ転送タイミングを制御します。これはUSBのAsyncモードと同様のジッター対策が元々組み込まれた設計です。

---

## サンプルレートの手動設定

通常はmoOdeが自動的にネイティブレートで出力しますが、必要に応じてFireface 400のサンプルレートを手動で変更できます。

以下のCプログラムをビルドしてください：

```bash
cat > ~/ff400_config.c << 'EOF'
#include <stdio.h>
#include <stdint.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <linux/firewire-cdev.h>
#include <endian.h>

#define FF400_STF 0x000080100500ULL

int main(int argc, char *argv[]) {
    if (argc != 2) {
        printf("Usage: %s <rate>\n", argv[0]);
        printf("  rate: 44100 48000 88200 96000 176400 192000\n");
        return 1;
    }

    int rate = atoi(argv[1]);
    int valid_rates[] = {44100, 48000, 88200, 96000, 176400, 192000};
    int valid = 0;
    for (int i = 0; i < 6; i++) {
        if (rate == valid_rates[i]) { valid = 1; break; }
    }
    if (!valid) { printf("Invalid rate: %d\n", rate); return 1; }

    printf("Setting STF rate=%d\n", rate);

    int fd = open("/dev/fw1", O_RDWR);
    if (fd < 0) { perror("open /dev/fw1"); return 1; }

    struct fw_cdev_event_bus_reset bus_reset;
    memset(&bus_reset, 0, sizeof(bus_reset));

    struct fw_cdev_get_info info;
    memset(&info, 0, sizeof(info));
    info.version = 4;
    info.bus_reset = (uint64_t)(uintptr_t)&bus_reset;

    if (ioctl(fd, FW_CDEV_IOC_GET_INFO, &info) < 0) {
        perror("FW_CDEV_IOC_GET_INFO");
        close(fd);
        return 1;
    }

    uint32_t data = htole32(rate);

    struct fw_cdev_send_request req;
    memset(&req, 0, sizeof(req));
    req.tcode      = TCODE_WRITE_QUADLET_REQUEST;
    req.length     = 4;
    req.offset     = FF400_STF;
    req.closure    = 0;
    req.data       = (uint64_t)(uintptr_t)&data;
    req.generation = bus_reset.generation;

    if (ioctl(fd, FW_CDEV_IOC_SEND_REQUEST, &req) < 0) {
        perror("FW_CDEV_IOC_SEND_REQUEST");
        close(fd);
        return 1;
    }

    printf("Done!\n");
    close(fd);
    return 0;
}
EOF
gcc -o ~/ff400_config ~/ff400_config.c
```

使い方：

```bash
~/ff400_config 44100   # 44.1kHzに設定
~/ff400_config 48000   # 48kHzに設定
~/ff400_config 96000   # 96kHzに設定
~/ff400_config 192000  # 192kHzに設定
```

---

## 注意事項

### カーネルアップデート時の再ビルドについて

Raspberry Pi OSのカーネルがアップデートされた場合、インストールしたモジュールは無効になります。カーネルの自動アップデートを停止しておくか、アップデート後に手順8〜13を再実施して再ビルドするかが必要となります。

カーネルバージョンが変わったかどうかは以下で確認できます：

```bash
uname -r
```

### カード番号について

Fireface 400のALSAカード番号は起動のたびに変わる場合があります。moOdeのWeb UIでOutput deviceを選択し直すことで`_audioout.conf`が自動更新されます。

### Fireface 400のクロック設定について

Fireface 400のクロックソースはMac/WindowsのTotalMixから設定してフラッシュに保存しておくことをお勧めします。Internalクロックで使用するのが最も安定しています。Wordクロックを使用する場合は、再生するファイルのサンプルレートに合わせてWordクロック側も手動で変更する必要があります。

### DSD再生について

Fireface 400はDoP非対応のため、DSDファイルはPCMに変換されて再生されます（192kHz/32bit）。

### サポートについて

本リポジトリは個人的な検証に基づくものです。動作を保証するものではなく、サポートは提供できません。自己責任でご使用ください。

---

## ハードウェア構成例

筆者の実装例です。

**ケース：** GeeekPi製メタルケース（Raspberry Pi 5用、アクティブクーラー付き、PCIe周辺機器ボード対応／Metal Case for Raspberry Pi 5, with Pi 5 Active Cooler, Support X1000/X1001/X1003/N04/N05 PCIe Peripheral Board）。WaveshareのHATとStarTechのMini PCIeカードを装着した状態で収納できることを確認しています。

**FireWireコネクタ部分：** Mini PCIeカードのFireWireコネクタはケース外に露出します。タカチなどの小型金属ケースを加工してコネクタ部分を保護することをお勧めします。

（写真は後日追加予定）

---

## クレジット

本修正の調査・実装・検証はAnthropic社のAIアシスタント「Claude」のサポートのもとで行いました。同様の課題を抱えている方はClaudeに相談することで解決の糸口が見つかるかもしれません。
