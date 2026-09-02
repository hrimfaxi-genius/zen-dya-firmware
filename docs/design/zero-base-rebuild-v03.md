# Design: ゼロベース再構築(v0.3安定版ブランチ)で左右ペアリング不良を切り分ける

Status: implemented(rebuild-stable-v0.3ブランチ、ユーザーによる確認待ち)
Owner: hrimfaxi-genius / Claude(調査・設計) / Codex(実装)

## 背景

デバッグログビルドの容量削減を重ねても、実機に書き込めない状態が続いている
(2026-09-03時点)。debug方式に固執するのをやめ、調査中に見つかった別の
有力な容疑者を検証する形でゼロベース(現行の複雑化したDYA構成を一旦外した
状態)から再構築する。

**新たに判明した事実**: `config/west.yml` は `zmk` を `cormoran/zmk` の
**浮動ブランチ `main+custom-studio-protocol`** にピン止めしている。しかし:

- DYA機能を含まない本家 `zen-firmware`(hrimfaxi-genius/zen-firmware、
  E24-GH/zen-firmware由来)は公式 `zmkfirmware/zmk` の **タグ `v0.3`** を使用。
- DYAのruntime-input-processorモジュール自身のREADMEが推奨する構成も
  `cormoran/zmk` の **タグ `v0.3+custom-studio-protocol`**(v0.3ベースの安定版)。
- **このリポジトリだけが** `main`(継続更新中の開発ブランチ)を使っており、
  それに合わせて `zmk-component-bmp-boost` も通常のリリースタグではなく
  「Zephyr 4対応のため」コミットSHA固定にされている。

左右ペアリング不良は `44b63d0`(Studio lock修正前)に戻しても再現しており、
Studio lock関連の変更が原因ではないと切り分け済み。一方 `main+custom-studio-protocol`
を使い続けている点はこれまで疑っていなかった。**開発版ブランチを使い続けている
こと自体がBLE split pairingの原因である可能性がある。**

## 目的

1. RuntimeInputProcessor(DYA機能)を一旦外した、flashに余裕のあるクリーンな
   構成でペアリングを再検証できるようにする。
2. `main+custom-studio-protocol` → `v0.3+custom-studio-protocol`(安定版)に
   切り替えて、これが左右ペアリング不良の原因かどうかを切り分ける。

## Owner decisions (do not re-litigate)

1. 現行 `zen-dya-firmware` リポジトリ内に新しいブランチ(例:
   `rebuild-stable-v0.3`)を作る。`main` ブランチ(現行の実績のある成果物)
   には一切手を加えない。
2. `config/west.yml` の `zmk` の revision を `cormoran/zmk` の
   `v0.3+custom-studio-protocol` に変更する(現行の `main+custom-studio-protocol`
   から)。`zmk-component-bmp-boost` の revision は `zen-firmware` と同じ
   `v0.2` タグに戻す(現行のコミットSHA固定は `main` ブランチ=新しいZephyr
   向けの対応だったため、v0.3系に戻すなら不要になるはず。もしビルドが
   通らなければCodexの判断でSHA固定に戻してよい)。
3. `zmk-module-runtime-input-processor` と `zmk-feature-custom-settings` の
   projectエントリは west.yml から削除する(DYA機能はこの段階では入れない)。
   他の依存(zmk-feature-status-led, zmk-driver-paw3222,
   zmk-feature-cdc-acm-bootloader-trigger, zmk-feature-non-lipo-battery-management,
   zmk-scroll-snap)は変更しない。
4. `boards/shields/zen/zen_right.conf` から、DYA専用の3行
   (`CONFIG_ZMK_RUNTIME_INPUT_PROCESSOR`,
   `CONFIG_ZMK_RUNTIME_INPUT_PROCESSOR_STUDIO_RPC`,
   `CONFIG_ZMK_LOW_PRIORITY_THREAD_STACK_SIZE=2048`)を削除する。
   Studio lock修正(`CONFIG_ZMK_STUDIO_LOCKING=y` と
   `CONFIG_ZMK_STUDIO_LOCK_BLE_DIRECT_ADVERTISING_ON_UNLOCK=y`)は**維持する**
   (これは検証済みの正しい修正なので元に戻さない)。`zen_left.conf` は
   変更不要(既にDYA関連の記述なし)。
5. `boards/shields/zen/zen_right.overlay` から DYA の runtime-input-processor
   ノード(`zen_pointer_runtime` / `zen_scroll_runtime`)を削除し、本家
   `zen-firmware` と同じ標準の入力処理(`zip_temp_layer` / スケーラーのみ)に
   戻す。具体的な置き換え内容は下記「実装手順」に正確な差分を記載する。
   `zen_left.overlay` は変更不要(元々DYA関連の記述なし)。
6. `build.yaml` は本家 `zen-firmware` の構成(トラックボール等の物理
   ハードウェア snippet はそのまま維持。これはDYAではなくZEN本体のハードウェア
   機能なので削除しない)に合わせつつ、診断用に `split-pairing-debug` snippet
   を使った debug アーティファクトも追加する(DYA関連コードが無くなるため
   flashに余裕ができ、今度こそ収まるはず)。具体的な内容は下記「実装手順」
   参照。
7. `config/keymap.keymap` は変更不要(runtime-input-processor関連の記述は
   元々含まれていない。`&studio_unlock` はStudio利用に必要なので残す)。

## 実装手順(Codex 向け・最小限、正確な差分)

1. `git checkout -b rebuild-stable-v0.3` で新ブランチを作成する。

2. `config/west.yml` を編集する:
   - `projects` の `zmk` エントリの `revision: main+custom-studio-protocol` を
     `revision: v0.3+custom-studio-protocol` に変更。
   - `projects` の `zmk-component-bmp-boost` エントリの
     `revision: 2f5567523b6f0bc39575d48ed746ed9d635edf8b` を `revision: v0.2` に変更。
   - `projects` から `zmk-module-runtime-input-processor` と
     `zmk-feature-custom-settings` の2エントリを削除する。
   - `remotes` の `cormoran` エントリは(`zmk` が引き続き使うので)残す。

3. `boards/shields/zen/zen_right.conf` から以下の3行を削除する:
   ```
   CONFIG_ZMK_RUNTIME_INPUT_PROCESSOR=y
   CONFIG_ZMK_RUNTIME_INPUT_PROCESSOR_STUDIO_RPC=y
   CONFIG_ZMK_LOW_PRIORITY_THREAD_STACK_SIZE=2048
   ```

4. `boards/shields/zen/zen_right.overlay` を以下の内容に置き換える
   (本家 `zen-firmware` の同ファイルと同一):
   ```
   #include "zen.dtsi"
   #include <input/processors.dtsi>
   #include <dt-bindings/zmk/input_transform.h>

   &pointing_listener {
       input-processors =
           <&zip_xy_transform (INPUT_TRANSFORM_X_INVERT | INPUT_TRANSFORM_Y_INVERT)>,
           <&zip_temp_layer 1 500>;

       scroller {
           layers = <4>;
           input-processors =
               <&zip_xy_transform (INPUT_TRANSFORM_Y_INVERT)>,
               <&zip_xy_to_scroll_mapper>,
               <&scroll_scaler 1 28>;
           process-next;
       };
   };

   &zen_transform {
       col-offset = <7>;
   };
   ```

5. `build.yaml` を以下の内容に置き換える(本家 `zen-firmware` の構成 +
   診断用debugアーティファクトの追加):
   ```yaml
   include:
     - board: bmp_boost
       shield: zen_right
       snippet: "studio-rpc-usb-uart split-central input-trackball-pmw3610 input-listener input-split-listener-left-all"
       artifact-name: zen_right_trackball_pmw3610_central
     - board: bmp_boost
       shield: zen_right
       snippet: "zmk-usb-logging split-pairing-debug split-central input-trackball-pmw3610 input-listener input-split-listener-left-all"
       artifact-name: zen_right_trackball_pmw3610_central_debug
     - board: bmp_boost
       shield: zen_left
       snippet: "studio-rpc-usb-uart"
       artifact-name: zen_left_peripheral
     - board: bmp_boost
       shield: settings_reset
   ```
   (`snippets/split-pairing-debug/` は既存のものをそのまま使う。変更不要。)

6. コミットしてpushする。ローカルビルド確認・GitHub Actionsの結果待ちは
   行わない(方針は `codex-collaboration-workflow.md` のトークン節約ルール
   通り)。

7. 本ファイルの `Status:` 行を
   `implemented(rebuild-stable-v0.3ブランチ、ユーザーによる確認待ち)` に更新する。

## Files touched

- `config/west.yml`
- `boards/shields/zen/zen_right.conf`
- `boards/shields/zen/zen_right.overlay`
- `build.yaml`

## 次のステップ(このドキュメントの対象外・後続タスク)

1. ユーザーが `rebuild-stable-v0.3` ブランチのCI成功を確認し、
   `zen_right_trackball_pmw3610_central.uf2` / `zen_left_peripheral.uf2`
   (通常版、debugではない方)を実機に書き込んで左右ペアリングを確認する。
   - **直る場合**: `main+custom-studio-protocol`(浮動ブランチ)が左右
     ペアリング不良の原因だった可能性が高い。今後の本流はこのブランチを
     ベースにし、DYA機能(runtime-input-processor)を
     `v0.3+custom-studio-protocol` 前提で改めて追加し直す(別の設計
     ドキュメントで扱う)。
   - **直らない場合**: ソフトウェア(このプロジェクトの変更)が原因では
     なく、ハードウェアまたはより根本的な環境要因を疑う必要がある。この
     場合は `zen_right_trackball_pmw3610_central_debug.uf2` (今度はDYA
     関連コードが無いため書き込めるはず)でログを採取し、
     `docs/design/split-pairing-debug-logging.md` の手順で原因を特定する。
2. どちらの場合も、原因が確定次第、DYA機能の再統合(`zmk-module-runtime-input-processor`
   を `v0.3+custom-studio-protocol` ベースで正しく追加し直す)を次の設計
   ドキュメントとして起こす。
