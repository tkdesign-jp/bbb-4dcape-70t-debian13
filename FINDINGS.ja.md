# 解析で分かったこと(背景・発見集)

今回の解析の道のりで分かったこと全部。文脈が欲しい人向け — **README** だけ読めばレシピは追える、この文書は*なぜ*そのレシピなのかを語る。

> 🇬🇧 English version: [FINDINGS.md](FINDINGS.md)

---

## 1. 時系列 — ここに至るまで

| 年 | 出来事 |
|---|---|
| 2011 | BeagleBoard.org 財団が BeagleBone(白)を発売 |
| 2013 | **BeagleBone Black** リリース — TI AM335x Cortex-A8 720 MHz、512 MB RAM、4 GB eMMC、$45 |
| 2014-02 | **4D Systems が 4DCAPE-70T データシート(rev 1.2)を発行**。動作確認は **Angstrom 2013.06.20 + カーネル 3.8.13-bone37** 想定 |
| ~2015-2016 | **Angstrom ディストロが事実上 EOL**。後継は Robert C. Nelson がメンテする Debian image |
| 2017-2020 | Debian 9 / 10(カーネル 4.9 / 4.19)— stock dtbo で LCD7 cape が箱出し動作。cape の黄金時代 |
| 2020-2022 | Debian 11(カーネル 5.10)— まだ動作、「既知良好構成」とされる |
| 2023-2024 | Debian 12(カーネル 6.1)— 動作するが端境期、エッジケース出始める |
| 2025-2026 | **Debian 13(カーネル 6.12)— LCD7 cape が静かに動かなくなる**。DRM connector 無し、backlight 無し、画面真っ暗 |
| 2026-06-05 | Colin Bester さんが同じ問題(LCD4 cape の方)についてフォーラムスレッドを開く |
| 2026-06-15 | Colin の `panel-dpi` + OF-graph 修正(LCD4 用)が Robert C. Nelson によって upstream merge される |
| **2026-06-28** | **本リポが LCD7 + カーネル 6.12 では panel-dpi 修正だけでは足りないことを記録 — ただしカーネルダウングレード回避策で 30 分で完全復活** |

パターン: **cape 自体は何も変わっていない。変わったのはカーネル**。2013 年設計の、カーネル 3.8 の慣習に基づいたハードウェアは、その慣習が維持される間は動き続けた。カーネルが `tilcdc` と `pinctrl-single` を新しい DRM 標準に書き換えた時、cape の stock オーバーレイは静かにサポート対象から外れた。

---

## 2. カーネル 5.10 と 6.12 の間で何が壊れたか

### 2.1. `tilcdc` がレガシー panel binding を受け付けなくなった

stock の `BB-BONE-LCD7-01-00A3.dts` は panel をこう記述している:

```dts
panel {
    compatible = "ti,tilcdc,panel";
    panel-info {
        ac-bias = <255>;
        bpp = <16>;
        ...
    };
    display-timings {
        ...
    };
};
```

これは **レガシー `ti,tilcdc,panel` binding** で、3.x / 4.x 時代に TI LCDC コントローラ用 DPI パネルを記述する標準的な方法だった。

カーネル 6.x では、`tilcdc` DRM ドライバは **モダンな OF-graph + `panel-dpi`** binding を期待する:

```dts
&lcdc {
    port {
        lcdc_0: endpoint {
            remote-endpoint = <&panel_0>;
        };
    };
};

panel: panel {
    compatible = "panel-dpi";
    panel-timing { ... };
    port {
        panel_0: endpoint {
            remote-endpoint = <&lcdc_0>;
        };
    };
};
```

レガシー binding は静かに DRM connector を生成しない。カーネルは `tilcdc: no encoders/connectors found` をログし、画面は真っ暗のまま。

Robert C. Nelson は `CONFIG_DRM_TILCDC_PANEL_LEGACY=y` を露出して自身のビルドではレガシー binding を維持しているが、upstream の `bb.org-overlays` ソース自体は新 binding に更新されておらず、2026 年半ば時点でも `// FIXME - LCD doesn't init` というコメントと共に OF-graph endpoint をコメントアウトしたままになっている。

### 2.2. `pinctrl-single` のサブノードが scan されない

別の、より obscure な不具合: カーネル 6.12(本作業で経験的に確認)では `pinctrl-single` ドライバがこう報告する:

```
pinctrl-single 44e10800.pinmux: no pins entries for pinmux_bb_lcd_pwm_backlight_pins
```

…オーバーレイのコンパイル済み `.dtbo` には正しい pin entries が入っているにもかかわらず:

```
pinmux_bb_lcd_pwm_backlight_pins {
    pinctrl-single,pins = <0x48 0x06>;
    phandle = <0x01>;
};
```

(0x48 = `gpmc_a2` の pin offset、0x06 = MUX_MODE6 + PULLDOWN、すなわち P9_14 上の EHRPWM1A = バックライト PWM pin。)

dtbo は正しい。ドライバが拾わない。これがカーネル側の regression なのかオーバーレイ適用順序の問題なのかは完全には診断していない — そしてその必要も無かった、カーネル 5.10 にはこの問題がないから。

### 2.3. 結果: カスケード崩壊

両方の regression が同じ袋小路に流れ込む:

```
DRM connector 無し  →  ディスプレイパス無し
pinmux 未適用     →  PWM 起動せず  →  backlight 無し  →  panel probe 永遠に defer
```

片方だけ修正(例: panel-dpi で connector 解決)しても、もう片方(pinmux)が動かないと panel は defer 状態のまま。両方の層が動く必要がある — それが 5.10 では成立している。

---

## 3. なぜカーネル 5.10 が正解だったか

BeagleBoard.org の apt リポジトリから利用可能だったのは:

```
5.4, 5.10, 5.15, 6.12
```

(この image には 6.18 は無い)。判断の根拠:

- **5.4** — より古く、メンテも消極的。
- **5.10** — LTS、upstream で 2026 年 12 月までメンテ、Robert の apt フィードにも残ってる。cape を壊した tilcdc / pinctrl-single の書き換えより前の世代。
- **5.15** — 破綻ラインに近い。未検証。
- **6.12** — 本 cape で動かないことが確認済み(元の問題)。

5.10 は **本ハードウェア組み合わせで「確実に動く最新カーネル」**。Debian 11(Bullseye)と同じカーネルなので、BBB cape との動作実績がコミュニティに長年蓄積されている。

### 重要な Linux の事実: カーネル ABI は安定

Debian 13.5 のユーザーランド(glibc 2.40, systemd 256 等)がカーネル 5.10 で完璧に動くのは、**Linus Torvalds が 20 年以上強制している**から:「**ユーザースペースを壊すな**」。カーネル-ユーザースペース境界は安定 API。カーネル-ドライバ境界は安定じゃない。

これが本手順が機能する理由: カーネル(ハードウェアと話す、cape regression が住んでいる場所)をダウングレードしつつ、ユーザーランド(アプリと話す、欲しいセキュリティ修正がある場所)を最新に保てる。

---

## 4. これを可能にした人々

本リポは年単位の他人の労作の上に乗った薄いラッパー。

### Robert C. Nelson — [`@RobertCNelson`](https://forum.beagleboard.org/u/RobertCNelson)

[bb-kernel](https://github.com/RobertCNelson/bb-kernel) と BeagleBoard.org Debian image のメンテナ。地球上のほぼ全ての BeagleBone Linux ユーザーが、それと意識せず apt 経由で彼のカーネルを動かしている。BBB エコシステム全体の Linux story における単一障害点(良い意味で — 善意の独裁者)。

ほとんどのディストロが切り捨てた何年も後でも、彼は `bbb.io-kernel-5.10-bone` を BeagleBoard.org apt フィードに *メンテし続けて* いる。彼がそれをやっているから、本回避策が成立している。

### Colin Bester — [`@Colin_Bester`](https://forum.beagleboard.org/u/Colin_Bester)

2026 年 6 月、Colin は関連する 4DCAPE-43T(480×272)を Debian 13.5 + カーネル 6.18 で動かす方法について、5 層構造の緻密な解説を BeagleBoard フォーラムに投稿した。彼の診断が我々のぶつかった問題に名前を与えた: レガシー `ti,tilcdc,panel` binding が新カーネルで静かに失敗する。彼の `panel-dpi` 修正は LCD7 + カーネル 6.12 では十分でなかった(我々はさらに pinctrl-single 層と戦う必要があった)ものの、彼のスレッドが「これは現実のバグでユーザーエラーじゃない」と確信させてくれた。

ソース: [4DCape LCD on BeagleBone Black Debian 13.5 (v6.18.x)](https://forum.beagleboard.org/t/4dcape-lcd-on-beaglebone-black-debian-13-5-2026-05-19-iot-v6-18-x/44044)

### 4D Systems(オーストラリア)

4DCAPE-70T を設計・販売。2014 年のデータシート(rev 1.2、11 ページ)で、本 cape が Angstrom 2013.06.20 向けに開発されたこと、それ以降のソフトウェア変更について 4D Systems は責任を負わないことを率直に明記している。この誠実さは尊敬に値する — 良いハードウェアを出荷し、その前提を明確に文書化した。データシートで明記されている CircuitCo LCD7-00A3 ドライバの再利用が、bb.org-overlays プロジェクトが両 cape 共用の単一 dtbo を提供できる理由でもある。

### CircuitCo

オリジナルの LCD7-00A3 cape を設計、4DCAPE-70T はそのソフトウェア互換版。BBB 時代の CircuitCo の dtbo 作業無しに、すべての出発点が無かった。

### `bb.org-overlays` プロジェクト

[github.com/beagleboard/bb.org-overlays](https://github.com/beagleboard/bb.org-overlays) — すべての cape のデバイスツリーオーバーレイの中央リポジトリ。BeagleBoard コミュニティでメンテされている。本書で使う stock `BB-BONE-LCD7-01-00A3.dtbo` はここから来ている、無修正で。

### そして広いハードウェアハッキング文化から

哲学的に **Trammell Hudson さん**([`@osresearch`](https://github.com/osresearch))に敬意を — Magic Lantern、Heads、Thunderstrike、hcpy、papercraft。今回の作業に直接関わってはいないが、「**ベンダーがサポートを止めたものを分解し、現代世界で動かし続ける**」という発想自体が、本リポを可能にした本能と同じ。BBB + 4DCAPE-70T は精神的にはこの伝統への小さな一文。

---

## 5. うまくいかなかった試み(後から来る人が省けるように記録)

| 試したアプローチ | 結果 |
|---|---|
| stock Debian 13.5 + stock `BB-BONE-LCD7-01-00A3.dtbo` | 画面真っ暗、`no encoders/connectors found` |
| `am33xx_pwm-00A0.dtbo` 追加 — PWM 親有効化 | `pwmchip0–3` は出るが、`48302200.pwm`(EHRPWM1)は bind しない。backlight も無し |
| `BB-PWM1-00A0.dtbo` 追加(PWM1 個別) | 同じ — EHRPWM1 は bind しない。`pinmux_bb_lcd_pwm_backlight_pins` の「no pins entries」警告も残る |
| `enable_uboot_cape_universal=1` | LCD7 パスは解決しない。cape-universal は *別の* pin 割り当てスキームで LCD7 オーバーレイと衝突する |
| GPIO22 を sysfs から手動 export してバックライト high 強制 | `Device or resource busy` — panel ドライバが既に P9_14/GPIO22 を確保している(実際に backlight を駆動していないにもかかわらず) |
| LCD7 dtso を自前で `panel-dpi` + OF-graph に書き換え(Colin のパターンを LCD7 に移植) | `no encoders/connectors found` は突破 — `card0-LVDS-1` が `connected` になり `/dev/fb0` も作られる。(余談: ここでは connector 名が `LVDS-1` になるが、カーネル 5.10 + stock dtbo では `DPI-1` になる。tilcdc ドライバが connector 名を panel binding の宣言形式から拾うため、カーネル/binding の組み合わせで *同じ* DPI 出力でも異なる名前になる。)だがカーネル 6.12 では `pinmux_bb_lcd_pwm_backlight_pins` が pin を pick up しないので backlight は起動せず、panel probe も defer したまま |
| `sudo apt-get dist-upgrade`(フォーラムで Robert が推奨) | 6.12.x シリーズ内のマイナー上げのみ(`6.12.90 → 6.12.93-bone62`)。この image では 6.18.x に上がらない |
| `disable_uboot_overlay_video=1` | 必要な補助修正 — これをしないと BBB 内蔵 TDA998x HDMI トランスミッタが `&lcdc` を先に取得してパネル接続を奪う。カーネルダウングレードと組み合わせるとクリーンに動く |

学んだこと: カーネル 6.12 では、**オーバーレイのどんな組み合わせをいじっても pinctrl-single の regression を突破できない**。カーネル本体にパッチを当てるか、カーネルを変えるかの 2 択。我々は後者を選んだ。

---

## 6. ハードウェア面で判明した事実

| 事実 | 詳細 |
|---|---|
| 4DCAPE-70T は CircuitCo LCD7-00A3 の 4D Systems ブランド版 | EEPROM は cape を `BB-BONE-LCD7-01 / 00A3` として識別 — BBB はどちらでも同じオーバーレイを load する |
| LCD パネルは **ThreeFive S9700RTWV35TR**(または互換: パネル裏面シルクの `RF110-XSD-7.0`) | 800×480、30 MHz ピクセルクロック、24-bit パネルだが 16bpp RGB565 駆動 |
| バックライトは **P9_14 上の EHRPWM1A で PWM 制御**(AM335x レジスタ 0x48302200) | この pin に `pinctrl-single` で MUX_MODE6 を適用する必要がある |
| LCD 表示領域: 154.1 mm × 85.9 mm | 4D Systems 機構図より |
| 4DCAPE-70T はそれなりの電流を引く — **5V/2A 外部 DC アダプタが必要** | USB 給電(500 mA)だけでは LCD バックライトを安定駆動できない |
| **EEPROM ジャンパは両方クローズが必要** | 4D Systems データシートより、さもなくば cape ID 検出に失敗することがある |
| BBB 内蔵 HDMI トランスミッタ(**NXP TDA998x**)が LCDC を共有 | HDMI 仮想 cape を無効化しないと、tilcdc がパネルではなく tda998x に bind してしまう |
| GPIO22 = P9_14 = バックライト enable | panel ドライバがこの pin を確保した状態だと、`/sys/class/gpio/` から手動 export 不可 |

---

## 7. 学び(メタな教訓)

### 7.1. 古いハードはカーネルと共に年を取る

2013-2014 年に設計されたハードウェアアクセサリは全て、「これが今日の Linux ドライバのあり方だ」という暗黙の前提を持っている。10 年後にカーネルがそれを書き換えた時、ハードウェアが壊れたわけではない — その *ドライバの繋ぎ目* が壊れたのである。ハードウェアはずっと健全だった。

使い続けたい obsolete なハードウェアにとっては、**走らせるカーネルもハードウェア互換性リストの一部**、cape そのものと同じくらいに。

### 7.2. ユーザースペース安定性は贈り物、その使えるオプションを守れ

Linus の「ユーザースペースを壊すな」ルールが、我々が *カーネルだけを* ダウングレードできた理由。他のほとんどの OS ではこれは不可能 — カーネルとユーザーランドはセットで出荷される。Linux では数世代の範囲内なら炸裂せず混ぜ合わせられる。この自由を使え。

### 7.3. 個人メンテナはインフラ

本修正が数日のポーティング作業ではなく 30 分の演習で済むのは、Robert C. Nelson が `bbb.io-kernel-5.10-bone` をメンテし、apt 経由で到達可能にし続けているから。彼が止めたら、一夜にしてこれが大変な作業になる。Robert のような人々への支援を検討するべき — 彼らは一人でエコシステム全体を支えている。

### 7.4. 「表示されない」は「動いてない」じゃない

地味だが大事。LCD が点く前から `/dev/fb0` は機能していた。カーネルは何の問題も無かった。cape の *ディスプレイパス* が壊れていただけ。診断時に混同するな — 「フレームバッファ存在」「パネル信号生成」「バックライト点灯」「ピクセル視認」を分離して切り分けること。

### 7.5. 粘りは triage に勝つ

この debug セッションのいくつかの局面で、合理的エンジニアリング判断は「カーネル 6.12 パスは死亡宣告、既知良好な古い image に切り替える」だった。それを追求し続けてカーネルだけのダウングレード解を見つけたことが、将来同じ問題に当たる人の時間を毎回 何時間も節約する。エンジニアリング的合理(損切り、フォールバック)と正しい答え(「あと一つ試す事がある」)が分岐することがある。

---

## 8. 更に学ぶには

- **BeagleBone Black wiki**: [elinux.org/Beagleboard:BeagleBoneBlack](https://elinux.org/Beagleboard:BeagleBoneBlack)
- **Cape デバイスツリーオーバーレイドキュメント**: [elinux.org/Beagleboard:BeagleBoneBlack_Debian#U-Boot_Overlays](https://elinux.org/Beagleboard:BeagleBoneBlack_Debian#U-Boot_Overlays)
- **bb.org-overlays リポ**: [github.com/beagleboard/bb.org-overlays](https://github.com/beagleboard/bb.org-overlays)
- **Robert C. Nelson の bb-kernel**: [github.com/RobertCNelson/bb-kernel](https://github.com/RobertCNelson/bb-kernel)
- **BeagleBoard フォーラム**(Colin のスレッドがある場所): [forum.beagleboard.org](https://forum.beagleboard.org)
- **Linux カーネル `tilcdc` ドライバソース**: 任意のカーネルツリーの `drivers/gpu/drm/tilcdc/`
- **`pinctrl-single` binding ドキュメント**: `Documentation/devicetree/bindings/pinctrl/pinctrl-single.txt`
- **4D Systems 4DCAPE-70T データシート**: [resources.4dsystems.com.au/datasheets/cape/4DCAPE-70T/](https://resources.4dsystems.com.au/datasheets/cape/4DCAPE-70T/)
- **Linus による「ユーザースペースを壊すな」**(canonical な rant): [lkml.org/lkml/2012/12/23/75](https://lkml.org/lkml/2012/12/23/75)

---

*本文書は 1 日の午後の debug で学んだことの記録 — 2026-06-28、熊本、日本。修正と追加歓迎、pull request で。*
