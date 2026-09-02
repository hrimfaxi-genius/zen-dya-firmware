# Design: ゼロベース再構築(v0.3安定版ブランチ)で左右ペアリング不良を切り分ける

Status: implemented(v3ブランチ、ユーザーによる確認待ち)
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

1. 現行 `zen-dya-firmware` リポジトリ内に新しいブランチ `v3` を作る。
   `main` ブランチ(現行の実績のある成果物)には一切手を加えない。
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

1. `git checkout -b v3` で新ブランチを作成する。

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
   `implemented(v3ブランチ、ユーザーによる確認待ち)` に更新する。

## Files touched

- `config/west.yml`
- `boards/shields/zen/zen_right.conf`
- `boards/shields/zen/zen_right.overlay`
- `build.yaml`

## 追加修正(Codex向け・ビルドエラーの修正)

`v3`(実装時は`rebuild-stable-v0.3`という名前でCodexが作成)ブランチの
GitHub Actions手動実行(Run #16)で、`zen_right`のビルドが2つとも
(通常版・debug版)以下のエラーで失敗した:

```
devicetree error: /pointing_listener: undefined node label 'zen_pointer_runtime'
```

原因は、当初の実装手順で見落としていたファイルがあったため。
`boards/shields/zen/zen_right.overlay` は指示通り修正されたが、
`snippets/input-trackball-pmw3610/input-trackball-pmw3610.overlay`
(トラックボール用snippet、build.yamlの`zen_right`アーティファクトで
使われている)が `&pointing_listener` を**再度上書き**しており、
そちらがまだ削除したはずの `&zen_pointer_runtime` / `&zen_scroll_runtime`
を参照したままだった。同様に
`snippets/input-split-listener-left-all/input-split-listener-left-all.overlay`
も独自に `zen_left_pointer_runtime` / `zen_left_scroll_runtime`
(`compatible = "zmk,input-processor-runtime"`)を定義しており、
`zmk-module-runtime-input-processor` をwest.ymlから外すとこのcompatibleの
バインディングが見つからずビルドが失敗する見込み。

以下2ファイルを、本家 `zen-firmware` の同名ファイルと**完全に同じ内容**に
置き換える:

1. `snippets/input-trackball-pmw3610/input-trackball-pmw3610.overlay`:
   ```
   #include <dt-bindings/zmk/input_transform.h>

   / {
       pmw3610_scroll_scaler: pmw3610_scroll_scaler {
           compatible = "zmk,input-processor-scaler";
           #input-processor-cells = <2>;
           type = <INPUT_EV_REL>;
           codes = <INPUT_REL_WHEEL INPUT_REL_HWHEEL>;
           track-remainders;
       };
   };

   &spi0 {
       status = "okay";
       compatible = "nordic,nrf-spim";
       pinctrl-0 = <&spi0_default>;
       pinctrl-1 = <&spi0_sleep>;
       pinctrl-names = "default", "sleep";
       cs-gpios = <&gpio0 20 GPIO_ACTIVE_LOW>;

       trackball: pointing_device: pointing_device@0 {
           status = "okay";
           compatible = "pixart,pmw3610-alt";
           reg = <0>;
           spi-max-frequency = <2000000>;
           irq-gpios = <&gpio0 19 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>;
           cpi = <400>;
           evt-type = <INPUT_EV_REL>;
           x-input-code = <INPUT_REL_X>;
           y-input-code = <INPUT_REL_Y>;
           force-awake;
       };
   };

   &pointing_listener {
       input-processors =
           <&zip_xy_transform (INPUT_TRANSFORM_XY_SWAP)>,
           <&zip_temp_layer 1 500>;

       scroller {
           layers = <4>;
           input-processors =
               <&zip_xy_transform (INPUT_TRANSFORM_XY_SWAP | INPUT_TRANSFORM_Y_INVERT)>,
               <&zip_xy_to_scroll_mapper>,
               <&pmw3610_scroll_scaler 1 4>;
           process-next;
       };
   };
   ```

2. `snippets/input-split-listener-left-all/input-split-listener-left-all.overlay`:
   ```
   #include <dt-bindings/zmk/input_transform.h>

   / {
       left_paw3222_scroll_scaler: left_paw3222_scroll_scaler {
           compatible = "zmk,input-processor-scaler";
           #input-processor-cells = <2>;
           type = <INPUT_EV_REL>;
           codes = <INPUT_REL_WHEEL INPUT_REL_HWHEEL>;
           track-remainders;
       };

       left_pmw3610_scroll_scaler: left_pmw3610_scroll_scaler {
           compatible = "zmk,input-processor-scaler";
           #input-processor-cells = <2>;
           type = <INPUT_EV_REL>;
           codes = <INPUT_REL_WHEEL INPUT_REL_HWHEEL>;
           track-remainders;
       };

       left_trackpad_scroll_scaler: left_trackpad_scroll_scaler {
           compatible = "zmk,input-processor-scaler";
           #input-processor-cells = <2>;
           type = <INPUT_EV_REL>;
           codes = <INPUT_REL_WHEEL INPUT_REL_HWHEEL>;
           track-remainders;
       };

       split_inputs {
           #address-cells = <1>;
           #size-cells = <0>;

           left_trackball_pmw3610_split: left_trackball_pmw3610_split@10 {
               compatible = "zmk,input-split";
               reg = <10>;
           };

           left_trackball_paw3222_split: left_trackball_paw3222_split@11 {
               compatible = "zmk,input-split";
               reg = <11>;
           };

           left_trackpad_split: left_trackpad_split@12 {
               compatible = "zmk,input-split";
               reg = <12>;
           };
       };

       left_trackball_pmw3610_listener: left_trackball_pmw3610_listener {
           compatible = "zmk,input-listener";
           device = <&left_trackball_pmw3610_split>;
           status = "okay";
           input-processors =
               <&zip_xy_transform (INPUT_TRANSFORM_XY_SWAP | INPUT_TRANSFORM_X_INVERT | INPUT_TRANSFORM_Y_INVERT)>,
               <&zip_temp_layer 1 500>;

           scroller {
               layers = <4>;
               input-processors =
                   <&zip_xy_scaler 1 56>,
                   <&zip_xy_transform (INPUT_TRANSFORM_XY_SWAP | INPUT_TRANSFORM_X_INVERT)>,
                   <&zip_xy_to_scroll_mapper>,
                   <&left_pmw3610_scroll_scaler 1 20>;
               process-next;
           };
       };

       left_trackball_paw3222_listener: left_trackball_paw3222_listener {
           compatible = "zmk,input-listener";
           device = <&left_trackball_paw3222_split>;
           status = "okay";
           input-processors =
               <&zip_xy_transform (INPUT_TRANSFORM_X_INVERT | INPUT_TRANSFORM_Y_INVERT)>,
               <&zip_temp_layer 1 500>;

           scroller {
               layers = <4>;
               input-processors =
                   <&zip_xy_scaler 1 56>,
                   <&zip_xy_to_scroll_mapper>,
                   <&left_paw3222_scroll_scaler 1 28>;
               process-next;
           };
       };

       left_trackpad_listener: left_trackpad_listener {
           compatible = "zmk,input-listener";
           device = <&left_trackpad_split>;
           status = "okay";
           input-processors =
               <&zip_xy_transform (INPUT_TRANSFORM_X_INVERT | INPUT_TRANSFORM_Y_INVERT)>,
               <&zip_temp_layer 1 500>;

           scroller {
               layers = <4>;
               input-processors =
                   <&zip_xy_scaler 1 960>,
                   <&zip_xy_transform (INPUT_TRANSFORM_Y_INVERT)>,
                   <&zip_xy_to_scroll_mapper>,
                   <&left_trackpad_scroll_scaler 1 28>;
               process-next;
           };
       };
   };
   ```

3. 念のため、リポジトリ全体を以下のキーワードでgrepし、他に
   `zen_pointer_runtime` / `zen_scroll_runtime` / `zen_left_pointer_runtime` /
   `zen_left_scroll_runtime` / `zmk,input-processor-runtime` /
   `RUNTIME_INPUT_PROCESSOR` を参照している箇所が残っていないか確認する
   (`zephyr/`, `dependencies/`, `build/` ディレクトリは除外してよい)。
   見つかった場合は同様に本家`zen-firmware`の対応ファイルと比較し、
   DYA関連の参照を削除する。
4. コミットしてpushし、`workflow_dispatch`で「Build ZEN DYA firmware」を
   `v3`(またはCodexが作成した実際のブランチ名)で再実行できる状態にする
   (実行自体はユーザーが行う。Codexは待たない)。

## 次のステップ(このドキュメントの対象外・後続タスク)

1. ユーザーが `v3` ブランチのCI成功を確認し、
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
