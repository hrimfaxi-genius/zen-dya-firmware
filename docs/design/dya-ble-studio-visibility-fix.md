# Design: DYA Studio に ZEN が表示されない問題の正式な修正

Status: implemented
Owner: hrimfaxi-genius / Claude(調査・設計) / Codex(実装)

## 背景・これまでの経緯（再度やり直さないこと）

`44b63d0`(Fix ZEN DYA CI dependencies)の時点:

- Windows と ZEN の通常の Bluetooth 接続(HID キーボードとしてのペアリング)は正常
- DYA Studio の「Connect Bluetooth」を押しても、端末一覧に ZEN が出てこない
  ← **これが元々の課題**

そこから3回の修正試行があったが、いずれも根本原因を解決できていない。

1. `a0cd002`「Enable DYA BLE management」: サードパーティ製の
   `zmk-module-ble-management` モジュールを追加し `CONFIG_ZMK_BLE_MANAGEMENT` /
   `CONFIG_ZMK_BLE_MANAGEMENT_STUDIO_RPC` を有効化。
   → **`062a864`「Restore proven Studio BLE configuration」で全て revert 済み。
   このアプローチは採用しない。** (このモジュールは Studio RPC の BLE
   トランスポートとは無関係な、split 接続/プロファイル管理用の別物だった)
2. `1ac89d3`「Improve Windows BLE pairing compatibility」:
   `CONFIG_BT_CTLR_PHY_2M=n` を追加。
   → **これを適用した状態で、今度は Windows との通常の Bluetooth 接続自体が
   完全にできなくなった(退行)。** 原因不明のまま現在ローカルの作業コピー上では
   このコミットの内容は打ち消され(下記参照)、まだ push されていない。

## 現在のローカル未コミット状態(Codex がトークン切れで中断した時点)

`C:\Users\hrimf\Documents\ZEN\zen-dya-framework` の未コミット差分
(HEAD=`1ac89d3` に対する working tree の変更、まだ commit していない):

- `boards/shields/zen/zen_right.conf`:
  - `CONFIG_BT_CTLR_PHY_2M=n` を削除(1ac89d3 の変更を打ち消し)
  - `CONFIG_ZMK_STUDIO_LOCK_BLE_DIRECT_ADVERTISING_ON_UNLOCK=y` を追加
  - `CONFIG_ZMK_STUDIO_LOCKING=n` は **そのまま残っている**(ここが未完了)
- `config/keymap.keymap`:
  - layer_3 の1番目のバインディングを `&trans` → `&studio_unlock` に変更

この2つの変更は方向性として正しい(下記「根本原因」参照)が、
**1行の変更漏れにより機能していない状態**で中断している。

## 根本原因の特定(cormoran/zmk フォーク本体のソースで確認済み)

`config/west.yml` で参照している `cormoran/zmk`
(branch: `main+custom-studio-protocol`) の
`app/src/studio/Kconfig` を実際に確認した(GitHub上で直接ソースを読んだ、
推測ではない)。該当箇所:

```kconfig
menuconfig ZMK_STUDIO_LOCKING
    bool "Lock Support"
...
config ZMK_STUDIO_LOCK_BLE_DIRECT_ADVERTISING_ON_UNLOCK
    bool "Enable Directed Advertising on Unlock"
    default y if ZMK_STUDIO_LOCKING && ZMK_BLE
    help
      When enabled, the keyboard will enable directed advertising to active profile
      during unlock. It's required to detect device from web bluetooth API for some browsers.
```

さらに:

```kconfig
menuconfig ZMK_STUDIO_RPC
    ...
    imply ZMK_STUDIO_LOCKING if !ARCH_POSIX
```

**つまり「Web Bluetooth API から端末を検出するには、`unlock` 時の directed
advertising が必要で、それは `ZMK_STUDIO_LOCKING=y` のときにしか意味を持たない」
と、cormoran/zmk 本体のヘルプテキストに明記されている。**

このプロジェクトは `44b63d0` の時点から一貫して
`CONFIG_ZMK_STUDIO_LOCKING=n`(ロック機能を無効化)にしていた
(おそらく unlock キー操作を毎回求められるのを避けるため)。しかし
これが原因で、`ZMK_STUDIO_LOCK_BLE_DIRECT_ADVERTISING_ON_UNLOCK` の
`default y if ZMK_STUDIO_LOCKING && ZMK_BLE` 条件が満たされず、directed
advertising が有効化されない → DYA (Web Bluetooth) が ZEN を発見できない、
という一連の流れになっていた可能性が高い。

**`CONFIG_ZMK_STUDIO_LOCKING=n` にしていたこと自体が、そもそもの
「DYA の端末一覧に出てこない」バグの根本原因だったと考えられる。**

## Owner decisions (do not re-litigate)

1. **`zmk-module-ble-management`(サードパーティモジュール)は使わない。**
   `062a864` の revert 済みの状態を維持する。
2. **`CONFIG_BT_CTLR_PHY_2M=n` は今回は追加しない。** Windows 接続を壊した
   直接の原因の疑いが強く、根本原因(ロック設定)とは無関係。もし
   `CONFIG_ZMK_STUDIO_LOCKING=y` 化後の実機テストで特定の Windows
   Bluetooth アダプタのペアリング不具合が再現した場合のみ、原因を切り分けた
   上で改めて検討する(セット でまとめて入れない)。
3. **`CONFIG_ZMK_STUDIO_LOCKING` を `n` → `y` に変更する。**
   既存の未コミット変更(`CONFIG_ZMK_STUDIO_LOCK_BLE_DIRECT_ADVERTISING_ON_UNLOCK=y`
   と keymap への `&studio_unlock` 追加)は正しい方向なので**そのまま活かす**。
   今回追加するのはこの1行だけで良いはず。
4. ロックを有効化する副作用として、DYA
   Studio に接続する前に毎回 `&studio_unlock`(layer_3 の左上、現在
   `&mo 3` で入るレイヤーの最初のキー)を押す必要がある
   (`CONFIG_ZMK_STUDIO_LOCK_ON_DISCONNECT`
   がデフォルト有効、`CONFIG_ZMK_STUDIO_LOCK_IDLE_TIMEOUT_SEC` はデフォルト
   600秒)。これは仕様であり不具合ではない。`web/src/App.tsx`
   側には既にこの操作を案内するヒント文言が実装済みなので、Web UI 側の
   追加対応は不要。
5. `boards/shields/zen/zen_left.conf`(peripheral側)への変更は不要と
   考えられる(Studio RPC・ロックは central 側だけで完結する)。ビルドが
   通ることと Renode テストが通ることで確認する。

## 実装手順(Codex 向け)

1. ローカルの未コミット差分(`zen_right.conf` の PHY_2M 削除 +
   `LOCK_BLE_DIRECT_ADVERTISING_ON_UNLOCK=y` 追加、`keymap.keymap` の
   `&studio_unlock` 追加)は**そのまま維持**する。破棄しない。
2. `boards/shields/zen/zen_right.conf` の
   `CONFIG_ZMK_STUDIO_LOCKING=n` を `CONFIG_ZMK_STUDIO_LOCKING=y` に変更する
   (これが今回追加で必要な唯一の変更点)。
3. `git diff` で最終的な差分が「(a) `CONFIG_BT_CTLR_PHY_2M=n` 削除、
   (b) `CONFIG_ZMK_STUDIO_LOCKING=n→y`、(c)
   `CONFIG_ZMK_STUDIO_LOCK_BLE_DIRECT_ADVERTISING_ON_UNLOCK=y` 追加、(d)
   keymap に `&studio_unlock` 追加」の4点のみであることを確認する。
4. `python -m unittest`(ルートの `test.py`)でビルド・Renode テストを実行し、
   成功を確認する。
5. README.md に、このプロジェクトでは Studio ロックを有効化しており、DYA
   に接続する前に `&studio_unlock` を押す必要があることを一言追記する
   (`062a864` で一度削除された BLE 関連の説明文言を、今回の正しい構成に
   合わせて書き直す形で良い)。
6. コミットメッセージは、これまでの3回の試行錯誤(a0cd002 / 062a864 /
   1ac89d3)の反省を踏まえた内容にする。例:
   `Fix Studio BLE advertising by enabling official lock/unlock flow`
7. push 後、GitHub Actions のビルドが成功することを確認する。
8. **実機での最終確認はユーザー(hrimfaxi-genius)が行う。** ビルドした
   UF2 を両手に書き込み → Windows と通常ペアリングできるか → `&studio_unlock`
   を押してから DYA の「Connect Bluetooth」で ZEN が一覧に出るか、の2点を
   手動でテストする必要がある(CI の Renode テストは実機の Windows
   ペアリング挙動までは検証できないため)。

## Files touched(想定)

- `boards/shields/zen/zen_right.conf`(既存の未コミット差分 + `LOCKING=n→y`)
- `config/keymap.keymap`(既存の未コミット差分をそのまま)
- `README.md`(ロック/unlock操作についての説明を1〜2文追記)

`src/studio/`, `proto/`, `web/` は変更不要(RPC プロトコル自体は無関係、
トランスポート層とロック設定だけの問題のため)。
