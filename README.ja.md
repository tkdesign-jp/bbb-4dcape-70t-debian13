# BBB + 4DCAPE-70T を Debian 13.5 (Trixie) で動かす

**4D Systems 4DCAPE-70T** 7インチ LCD cape を **BeagleBone Black** + **Debian 13.5 (Trixie)** で動作させる検証済み手順。**カーネルだけ `5.10-bone` LTS にダウングレード**し、ユーザーランドは Debian 13.5 のまま維持する方式。

> 🇬🇧 English version: [README.md](README.md)
> ⚠️ **いかなる手順を実行する前にも**、[DISCLAIMER.md](DISCLAIMER.md) を読むこと。特にカーネル 5.10 LTS が 2026 年 12 月でセキュリティサポート終了となる点について。
> 📖 背景、関係者の話、行き詰まった試行錯誤について読みたい場合は [FINDINGS.ja.md](FINDINGS.ja.md) を参照。

---

## この資料が解決する問題

新品の **BeagleBone Black** に公式 **Debian 13.5 (Trixie) base image (2026-05-19, カーネル 6.12.x)** を入れて 4DCAPE-70T を使おうとすると、LCD は真っ暗のままで以下のカーネルエラーが出る:

```
tilcdc 4830e000.lcdc: no encoders/connectors found
pinctrl-single 44e10800.pinmux: no pins entries for pinmux_bb_lcd_pwm_backlight_pins
platform backlight: deferred probe pending: platform: supplier 48302200.pwm not ready
```

根本原因:

1. 4DCAPE-70T はカーネル 3.8(Angstrom 2013)前提で設計されており、**レガシーな `ti,tilcdc,panel` デバイスツリーバインディング**を使っている。カーネル 6.x の `tilcdc` ドライバはこれを有効な DRM パネルと認識しなくなった。
2. カーネル 6.12 の `pinctrl-single` は、cape オーバーレイの pinmux サブノードを正しく解釈せず、結果として PWM バックライト用 pin が一切設定されない。
3. その結果、DRM connector が登録されず、`backlight` の sysfs デバイスも作られず、LCD は真っ暗のまま。

カーネル 6.18 用の LCD4 オーバーレイには有志による修正がある([Colin Bester さんの panel-dpi 書き換え](https://forum.beagleboard.org/t/4dcape-lcd-on-beaglebone-black-debian-13-5-2026-05-19-iot-v6-18-x/44044))。しかしカーネル 6.12 上の LCD7 では、`panel-dpi`/OF-graph 修正を移植しても `pinctrl-single` 層で別途詰む。

### 実際に動いた解決法

**動作中のカーネルを `bbb.io-kernel-5.10-bone` にダウングレードする**(Debian 13.5 ユーザーランドはそのまま維持)。カーネル 5.10 上では `bb.org-overlays` の **stock の `BB-BONE-LCD7-01-00A3.dtbo` が無修正で動く**、レガシー tilcdc バインディングも引き続き bind し、PWM バックライトもクリーンに起動する。

SD カードの焼き直しは不要、カーネルソースツリー作業も不要、オーバーレイへのパッチも不要。

---

## 検証済み構成

| 項目 | 値 |
|---|---|
| ボード | BeagleBone Black(TI AM335x Cortex-A8、512 MB RAM、rev C) |
| Cape | 4D Systems **4DCAPE-70T**(800x480、抵抗膜タッチ、ThreeFive S9700RTWV35TR / RF110-XSD-7.0 パネル) |
| Cape EEPROM | `BB-BONE-LCD7-01` rev `00A3` |
| OS イメージ | `am335x-debian-13.5-base-v6.12-armhf-2026-05-19-4gb.img.xz` |
| ユーザーランド | Debian 13.5 Trixie (armhf) |
| カーネル(修正後) | **5.10.x-bone**(`bbb.io-kernel-5.10-bone` 経由でインストール) |
| LCD オーバーレイ | `beagleboard/bb.org-overlays` の stock `BB-BONE-LCD7-01-00A3.dtbo` |
| 電源 | BBB の DC ジャックに 5V 2A AC アダプタ(USB 給電のみでは**不十分**) |
| 検証日 | 2026-06-28 |

手順実行後に動くもの:

- LCD バックライトが明るく安定して点灯
- カーネル `/dev/fb0` フレームバッファ存在(800x480, 16bpp RGB565)
- Linux TTY コンソール(ログインプロンプト)が LCD に表示される
- DRM connector `card0-DPI-1` が `connected` を報告(parallel RGB/DPI 出力、connector 名は tilcdc 由来で他ディストロでは多少違う場合あり)
- タッチパネルイベントが `/dev/input/event1` に来る(キャリブレーションは別ステップ)

---

## 前提条件

- BeagleBone Black に 4DCAPE-70T が物理装着済み(cape を BBB の上に重ね、EEPROM ジャンパーは出荷時のまま — 両方ショート)。
- BBB の DC ジャックに 5V 2A 電源を接続。
- microSD カード(4 GB 以上)に Debian 13.5 base image を焼いてある。
- 初期ネットワーク接続(有線 Ethernet が最も簡単、既知の動作する USB ドングル経由の Wi-Fi でも可)。
- SSH 用の別マシン(Mac / Linux / Windows)。3.3V USB-UART アダプタがあれば BBB のシリアルコンソール経由でも可。

---

## 手順

### 1. stock の Debian 13.5 image で起動

`am335x-debian-13.5-base-v6.12-armhf-2026-05-19-4gb.img.xz` を microSD カードに焼く(例: balenaEtcher)、電源 OFF の BBB に挿して起動。image は SD から動作し eMMC は無視されるので、SD を抜けばいつでも元に戻せる。

デフォルトログイン: `debian` / `temppwd`

### 2. ネットワーク接続を確保

有線 Ethernet + DHCP が最もシンプル。確認:

```bash
ip a
ping -c 3 8.8.8.8
```

Wi-Fi を使う場合、**先に既存のネットワークスタックを確認**してから NetworkManager を入れる:

```bash
# 現在何が動いているか
systemctl status systemd-networkd connman NetworkManager 2>/dev/null | grep -E "Active|Loaded"

# connman も NetworkManager も active でなければ NM をインストール
sudo apt update
sudo apt install -y network-manager
sudo nmcli device wifi connect "YOUR_SSID" password "YOUR_PASSPHRASE"
```

`connman` や別のスタックが既に active なら、そちらを使うか、NetworkManager を有効化する前に止めること — 2 つの network manager を同時に動かすとインターフェースが上下にバウンドし続ける。

(注意: 一部の古い USB Wi-Fi ドングルは WPA3 非対応 — ルーターが WPA2 SSID も流していればそちらを使う。)

### 3. 5.10-bone カーネルパッケージをインストール

```bash
sudo apt update
sudo apt install -y bbb.io-kernel-5.10-bone
```

これでカーネルと対応するデバイスツリーバイナリ、モジュールが `/lib/modules/5.10.x-bone/` 配下にインストールされる。6.12 カーネルは**削除されない** — 両方並列で残る。

パッケージインストール確認:

```bash
dpkg -l | grep "kernel-5.10-bone"
ls /lib/modules/ | grep "5\.10"
```

`/lib/modules/` 配下に `5.10.x-bone` ディレクトリが見えるはず。

### 4. U-Boot に 5.10 カーネルでブートするよう指示

**まず `/boot/uEnv.txt` のバックアップを取る**。ここでタイポすると BBB が起動しなくなり得る。バックアップがあれば、別マシンに SD カードを挿して `.bak` ファイルを戻すだけで即復旧できる:

```bash
sudo cp /boot/uEnv.txt /boot/uEnv.txt.bak
```

その後編集:

```bash
sudo nano /boot/uEnv.txt
```

ファイル冒頭付近のこの行を探す:

```
uname_r=6.12.xx-bone62
```

インストールした 5.10 のバージョンに変更。具体的な値はパッケージバージョンに依存するので、`ls /lib/modules/` で確認して置き換える。例:

```
uname_r=5.10.240-bone80
```

保存(`Ctrl+O`、`Enter`、`Ctrl+X`)。

### 5. 再起動して確認

```bash
sudo reboot
```

BBB が起動し直したら再 SSH、動作中カーネルを確認:

```bash
uname -r          # 5.10.x-boneXX が出るはず
```

### 6. LCD 確認

この時点で cape は既に動いている — 起動完了と同時にカーネル TTY ログインプロンプトが LCD に出る。

**オーバーレイがロードされる仕組み:** BBB の U-Boot は電源 ON 時に cape の I²C EEPROM を読み、`BB-BONE-LCD7-01` というボード ID を検出して、`/lib/firmware/` から対応する `BB-BONE-LCD7-01-00A3.dtbo` を自動でロードする。EEPROM が読める状態なら `/boot/uEnv.txt` の `uboot_overlay_addr*` 行を編集する必要は無い。確認方法:

```bash
sudo beagle-version | grep -i "loaded overlay"
# 期待: UBOOT: Loaded Overlay:[BB-BONE-LCD7-01-00A3] のような行が出る
```

(`beagle-version` は `bb-customizations` パッケージ由来で、BeagleBoard.org の Debian image にはプリインストールされているが、削ぎ落とした構成では入っていない場合がある。`command -v beagle-version` で何も出ない場合は以下の代替を使う:)

```bash
# /proc/device-tree から直接
ls /proc/device-tree/chosen/overlays/ 2>/dev/null
cat /proc/cmdline | tr ' ' '\n' | grep uboot_detected_capes
# あるいは dmesg
dmesg | grep -i "loaded overlay"
```

どの方法でも `BB-BONE-LCD7-01-00A3` が見えない場合、EEPROM が読めないか dtbo が `/lib/firmware/` に無い。その時は `/boot/uEnv.txt` で手動指定にフォールバック:

```
uboot_overlay_addr0=/lib/firmware/BB-BONE-LCD7-01-00A3.dtbo
```

sysfs での確認:

```bash
ls /dev/fb*                                  # /dev/fb0 が出るはず
ls /sys/class/backlight/                     # backlight デバイスが出るはず(本構成では "backlight" という名前)
ls /sys/class/drm/ | grep "^card0-"          # card0-DPI-1 が出るはず(tilcdc binding 依存で多少違う場合あり)
sudo cat /sys/class/drm/card0-*/status       # "connected" が出るはず
dmesg | grep -iE "tilcdc|panel|backlight"    # "no encoders/connectors found" が無いはず
```

手動バックライトテスト(`max_brightness` を先に読むこと — スケールはドライバ依存で必ずしも 0–100 ではない):

```bash
# デバイス固有の最大値を確認
cat /sys/class/backlight/*/max_brightness
# 本検証構成では 100 が出る

# 最大値で点灯(上記の値に合わせる)
echo 100 | sudo tee /sys/class/backlight/*/brightness

# 消灯
echo 0 | sudo tee /sys/class/backlight/*/brightness
```

(4DCAPE-70T + stock dtbo では `max_brightness` が 100 なので、たまたま `echo 100` で正しい。他 cape や別のオーバーレイでは 7、255、その他の値の可能性がある — 必ず先に読むこと。)

### 7. (任意) GUI を載せる

本ガイドの範囲外。フレームバッファが立ち上がっていれば、Debian 13 の標準 X11 スタックがそのまま動く — LXQT や openbox を入れる、または `/dev/fb0` 上で kiosk Chromium を動かすなど。

```bash
# 最小限の X + ウィンドウマネージャ(例)
sudo apt install -y xserver-xorg-core xserver-xorg-video-fbdev openbox xinit
```

---

## 6.12 カーネルに戻したい場合

両カーネルは並列インストールされている。6.12 に戻すには:

```bash
sudo nano /boot/uEnv.txt
# uname_r を元の 6.12.x-boneXX に戻す
sudo reboot
```

(LCD は再び真っ暗になる — これは 6.12 では期待通りの挙動。)

---

## なぜ特に 5.10 を選ぶか

この image・この日付時点で利用可能な BeagleBoard.org カーネルパッケージは以下:

```
bbb.io-kernel-5.4-bone
bbb.io-kernel-5.10-bone     ← 選択
bbb.io-kernel-5.15-bone
bbb.io-kernel-6.12-bone     ← デフォルト、本 cape で動かない
```

**5.10 は LTS** で、「まだメンテされている」かつ「本 cape を壊した tilcdc / pinctrl-single 書き換えより前の世代である」という両条件を最も良く満たす。5.4 はより古いが、アップデートも少なくなっている。5.15 は破綻ラインに近く、本書作者は未検証。

5.10 が EOL(現時点で 2026 年 12 月、DISCLAIMER 参照)に達して BeagleBoard.org apt フィードから削除される時が来たら、本ガイドは更新が必要になる。5.15-bone ブランチが代替として次の候補になる可能性がある。

---

## 謝辞

本書は他の人々の仕事の上に成立している:

- **[Colin Bester](https://forum.beagleboard.org/u/Colin_Bester) 氏** — LCD4 cape での panel-dpi / OF-graph 診断によって問題の名前を与えてくれた。結果としてその修正は LCD7 + カーネル 6.12 では十分でないと判明したが、診断自体が貴重だった。
- **[Robert C. Nelson](https://forum.beagleboard.org/u/RobertCNelson) 氏** — `bbb.io-kernel-5.10-bone` のメンテナンスと、複数カーネルを並列で保持する apt フィードを生かし続けてくれていること。これがカーネルだけのダウングレードを可能にしている。
- **[4D Systems](https://www.4dsystems.com.au/)** — 4DCAPE-70T ハードウェアと、本 cape が kernel 3.8 時代の Linux 向け設計であることを確認させてくれた 2014 年データシート。
- **[bb.org-overlays](https://github.com/beagleboard/bb.org-overlays)** プロジェクト — 適切なカーネル上では素直に動く stock の `BB-BONE-LCD7-01-00A3.dts`。
- **BeagleBoard フォーラムスレッド** [4DCape LCD on BeagleBone Black Debian 13.5 (v6.18.x)](https://forum.beagleboard.org/t/4dcape-lcd-on-beaglebone-black-debian-13-5-2026-05-19-iot-v6-18-x/44044) — これが現実の問題であることを確認させ、LCD4 側の修正を文書化してくれた。

本ガイドは上記のいずれの作業も改変していない。特定の現実世界の組み合わせを解決するために、それらをどう組み合わせるかを記述しているだけである。

---

## ライセンス

[MIT](LICENSE) — 知識は必要とする人のもの。無保証。カーネル 5.10 EOL の注意事項は [DISCLAIMER.md](DISCLAIMER.md) を参照。
