# Disclaimer

## English

This repository documents a known-working procedure for running the **4D Systems 4DCAPE-70T** display cape on **BeagleBone Black** with **Debian 13.5 (Trixie)**. Read this before following any step in this repository.

### Risk

Following these instructions involves:

- Installing an older Linux kernel package (5.10-bone) on a modern Debian 13.5 system, alongside the existing 6.12 kernel
- Modifying `/boot/uEnv.txt` to select which kernel boots (a misconfiguration can prevent boot)

Both are reversible operations — the original kernel remains installed and you can switch back by editing one line in `/boot/uEnv.txt`. The only way to get into trouble that requires re-flashing the SD card is to corrupt `/boot/uEnv.txt` itself; back it up first.

### Security consideration (READ THIS)

**The Linux kernel 5.10 is an LTS branch with security support officially ending in December 2026.** This procedure intentionally downgrades the kernel to obtain hardware compatibility with the LCD cape, at the cost of being on an older branch.

Implications:

- **For LAN-only / standalone / kiosk use** (the typical 4DCAPE-70T use case — a stand-alone display device), the security exposure is small: the attack surface is limited to whoever can reach the device on your local network. Many users of this cape never put the device on the internet at all.
- **For internet-facing or production use**, this configuration is **not recommended**. After December 2026, new kernel CVEs may not be back-ported to 5.10. If your device must remain network-exposed, plan to either:
  - Migrate the LCD cape function to different hardware that has current kernel support, or
  - Maintain the kernel yourself or via a paid LTS vendor, or
  - Accept the residual risk and isolate the device on a dedicated VLAN.

Robert C. Nelson (the BeagleBoard kernel maintainer) currently distributes `bbb.io-kernel-5.10-bone` via the standard apt repositories. As of this writing, updates within the 5.10.x series are still being delivered. Whether that continues past the upstream EOL is up to the maintainer.

### What is NOT addressed by this guide

- **Touch screen calibration** — the resistive touch panel works, but calibration is a separate topic (libinput `CalibrationMatrix` etc.).
- **X server / GUI desktop setup** — this guide focuses on getting the kernel console (`/dev/fb0`) on the LCD. GUI on top is a follow-up step (LXQT, Conky, kiosk Chromium, etc.).
- **Suspend/resume, audio cape coexistence, button gpio-keys mapping** — out of scope.

### Responsibility

By following any procedure in this repository, you accept that:

- **You are solely responsible** for any data loss, hardware damage, or other consequence.
- The author (**T.K DΞSIGN**) provides this material **as-is**, with **no warranty** of any kind, express or implied.
- The author has **no obligation** to provide support, fixes, or compensation if something goes wrong.
- This is a personal write-up of a working procedure on one specific hardware combination, shared as a reference. Your hardware may differ.

### Trademark notice

**BeagleBone**, **BeagleBone Black**, and **BeagleBoard** are property of BeagleBoard.org. References herein are descriptive only. **4D Systems**, **4DCAPE-70T**, and related marks are property of 4D Systems Pty. Ltd. This guide is not affiliated with, endorsed by, or sponsored by either organization.

---

## 日本語

このリポジトリは **4D Systems 4DCAPE-70T** ディスプレイ cape を **BeagleBone Black** + **Debian 13.5 (Trixie)** で動作させる検証済み手順を文書化したものです。本リポジトリ内のいかなる手順も、実行する前に必ず本文書を読むこと。

### リスク

本手順を実行することには以下が含まれる:

- 最新の Debian 13.5 システムに、既存の 6.12 カーネルと**並列で**古い Linux カーネル(5.10-bone)パッケージをインストールする
- `/boot/uEnv.txt` を編集してどちらのカーネルで起動するか選択する(誤設定で起動不能になり得る)

どちらも可逆操作 — 元のカーネルはインストールされたまま残り、`/boot/uEnv.txt` の 1 行を書き換えれば戻せる。SD カードの焼き直しが必要になる唯一のシナリオは `/boot/uEnv.txt` 自体を破壊した場合なので、先にバックアップを取ること。

### セキュリティに関する考慮(必読)

**Linux カーネル 5.10 は LTS ブランチで、公式セキュリティサポートは 2026 年 12 月で終了する**。本手順は意図的にカーネルをダウングレードして LCD cape との互換性を得ているが、その引き換えに古いブランチに乗っている。

含意:

- **LAN 内専用 / スタンドアロン / キオスク用途**(4DCAPE-70T の典型的な使い方 — 単体のディスプレイデバイス)であれば、セキュリティ的露出は小さい:攻撃面はローカルネットワーク内の到達可能者に限定される。本 cape ユーザーの多くは、そもそも当該機器をインターネットに繋がない。
- **インターネット公開 / 本番運用用途には推奨しない**。2026 年 12 月以降は、新規 kernel CVE が 5.10 にバックポートされない可能性がある。当該機器がネットワーク露出を維持する必要がある場合は、以下のいずれかを計画すること:
  - LCD cape 機能を、現行カーネルでサポートされる別ハードウェアに移行する
  - 自前で、あるいは有償 LTS ベンダー経由でカーネルメンテナンスする
  - 残留リスクを受容し、当該機器を専用 VLAN に分離する

Robert C. Nelson(BeagleBoard カーネルメンテナ)は現時点で `bbb.io-kernel-5.10-bone` を標準 apt リポジトリ経由で配布している。本文書作成時点で、5.10.x シリーズ内のアップデートは継続配信されている。これが上流 EOL 後も継続するかはメンテナの判断による。

### 本ガイドが扱わない範囲

- **タッチスクリーンキャリブレーション** — 抵抗膜タッチパネル自体は動作するが、キャリブレーションは別トピック(libinput `CalibrationMatrix` 等)。
- **X サーバ / GUI デスクトップ設定** — 本ガイドはカーネルコンソール(`/dev/fb0`)を LCD に出すところまでに集中する。GUI 化はフォローアップ(LXQT, Conky, kiosk Chromium 等)。
- **サスペンド / リジューム、オーディオ cape との同居、ボタンの gpio-keys マッピング** — 範囲外。

### 責任

本リポジトリ内のいかなる手順も、それを実行することで、利用者は以下に同意したものとする:

- **データ損失、ハードウェア損傷、その他あらゆる結果は利用者の自己責任**である。
- 作者(**T.K DΞSIGN**)は、明示・黙示を問わず**いかなる保証も伴わず現状渡し**で本リソースを提供する。
- 作者には、問題発生時のサポート、修正、補償の**いかなる義務も無い**。
- これは一つの特定ハードウェア構成における動作手順の個人記録であり、参考公開である。利用者のハードウェアは異なる可能性がある。

### 商標表記

**BeagleBone**、**BeagleBone Black**、**BeagleBoard** は BeagleBoard.org の所有である。本書中の言及は記述目的に限る。**4D Systems**、**4DCAPE-70T** および関連商標は 4D Systems Pty. Ltd. の所有である。本ガイドはいずれの組織とも提携・推薦・スポンサー関係に無い。
