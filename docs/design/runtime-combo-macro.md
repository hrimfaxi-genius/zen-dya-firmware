# Design: ランタイムコンボ(zmk-feature-runtime-combo)の追加

Status: design
Owner: hrimfaxi-genius / Claude(調査・設計) / Codex(実装)

## 背景

DYA Studio Web UIで「マクロコンボ」を開こうとすると、以下のエラーが表示される。

> このキーボードではランタイムコンボサブシステムを利用できません。
> `cormoran/zmk-feature-runtime-combo` がファームウェアに必要です。

これは `zmk-feature-runtime-combo`(Web UIから編集できるランタイムコンボを
追加するZMKモジュール)が本リポジトリに未導入なため。ユーザーからの要望で
今回追加する。

## 互換性の確認(調査済み、再度やり直さないこと)

`zmk-feature-runtime-combo` のREADME([cormoran/zmk-feature-runtime-combo](https://github.com/cormoran/zmk-feature-runtime-combo))に記載の必要な `zmk` ブランチは

```
revision: main+custom-studio-protocol
```

であり、これは本リポジトリの `config/west.yml` が現在使っている `zmk` の
revision(`main+custom-studio-protocol`)と**完全に一致する**。したがって
`zmk` のブランチ切り替えは不要。

依存する `zmk-feature-custom-settings` も、本リポジトリが既に
runtime-input-processor(DYA本体)の永続化バックエンドとして west.yml に
含めている(`revision: main`)。追加のプロジェクトエントリ変更は不要。

(参考: 別途要望のあった「BLE管理」(`zmk-module-ble-management`)は、
必要な `zmk` ブランチが `v0.3-branch+custom-studio-protocol+ble` で、
現在の `main+custom-studio-protocol` とは別系統のため、こちらは今回の
スコープに含めない。別の設計ドキュメントで扱う。)

## Owner decisions (do not re-litigate)

1. `config/west.yml` の `projects` に `zmk-feature-runtime-combo` を追加する。
   `zmk` エントリの revision は変更しない(既に一致しているため)。
2. `boards/shields/zen/zen_right.conf`(central側)にのみ設定を追加する。
   `zen_left.conf`(peripheral側)は、他のStudio RPC系機能
   (`CONFIG_ZMK_STUDIO_LOCKING`, `CONFIG_ZMK_RUNTIME_INPUT_PROCESSOR` 等)と
   同様に変更不要と考えられる(Studio RPC・ランタイム編集系の機能は
   central側だけで完結する、というこれまでのこのプロジェクトの構成に合わせる)。
3. `CONFIG_ZMK_STUDIO=y` は `zen_right.conf` に既に設定済みのため重複追加しない。
4. `CONFIG_ZMK_LOW_PRIORITY_THREAD_STACK_SIZE=2048` も `zen_right.conf` に
   既に設定済み(runtime-input-processorが要求する値と同じ)。
   `zmk-feature-runtime-combo` のREADMEも同じ値を要求しているため、
   これも変更不要(重複追加しない)。
5. コンパイル時デフォルトコンボ(`cormoran,runtime-combo-defaults` ノード)は
   **今回は追加しない**。まずはWeb UIからのランタイム編集のみで動作確認する
   (スコープを小さく保つ)。将来的にデフォルトコンボが欲しくなった場合は
   別途 `config/keymap.keymap` に追記する形で対応する。
6. flash容量については、過去にデバッグビルドで容量不足が発生した実績がある
   (`split-pairing-debug-logging.md` 参照)ため、CIビルドが成功することを
   もって確認とする。万一 `zen_right_trackball_pmw3610_central` の通常ビルドで
   容量不足が発生した場合は、その時点で対処法を別途検討する(今回は
   予防的な変更は行わない)。

## 実装手順(Codex 向け・最小限)

1. `config/west.yml` の `projects` に以下を追加する(`zmk-feature-runtime-combo`
   のREADMEに記載の設定そのまま):
   ```yaml
   - name: zmk-feature-runtime-combo
     remote: cormoran
     revision: main
     import: true
   ```
2. `boards/shields/zen/zen_right.conf` の末尾に以下を追加する:
   ```
   CONFIG_ZMK_RUNTIME_COMBO=y
   CONFIG_ZMK_RUNTIME_COMBO_STUDIO_RPC=y
   CONFIG_ZMK_STUDIO_RPC_RX_BUF_SIZE=128
   CONFIG_ZMK_STUDIO_RPC_CUSTOM_SUBSYSTEM_REQUEST_PAYLOAD_MAX_BYTES=96
   ```
   (`CONFIG_ZMK_STUDIO=y` と `CONFIG_ZMK_LOW_PRIORITY_THREAD_STACK_SIZE=2048` は
   既存の設定をそのまま使うため追加しない。)
3. コミットしてpushする。ローカルビルド確認・GitHub Actionsの結果待ちは
   行わない(`codex-collaboration-workflow.md` のトークン節約ルール通り)。
4. 本ファイルの `Status:` 行を `implemented(CI成功待ち/ユーザーによる実機確認待ち)`
   に更新する。

## Files touched

- `config/west.yml`
- `boards/shields/zen/zen_right.conf`

`zen_left.conf`, `config/keymap.keymap`, `src/`, `proto/`, `web/` は変更不要。

## 次のステップ(このドキュメントの対象外)

実機での最終確認はユーザー(hrimfaxi-genius)が行う。ビルドしたUF2を右手に
書き込み、DYA Studio Web UIで「マクロコンボ」のエラーが解消し、コンボの
追加・編集ができることを確認する。
