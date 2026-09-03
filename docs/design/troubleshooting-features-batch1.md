# Design: トラブルシューティング機能の追加(デバイス情報・ウォッチドッグ・キースイッチ診断・スタック使用量)

Status: implemented, CI成功(Build ZEN DYA firmware #25, commit d3cd831) / ユーザーによる実機確認待ち。詳細は末尾の「実装後の追記」参照。
Owner: hrimfaxi-genius / Claude(調査・設計) / Codex(実装)

## 背景

DYA Studioの「トラブルシューティング」タブに以下5つの項目があり、うち4つは
未実装(対応するZMKモジュールが未導入)だった。ユーザーからの要望で、
リスクの低い4つをまとめて実装する(5つ目の「トラックボールセンサー
(PMW3610)」は、既存のPMW3610ドライバの完全な置き換えが必要かつ開発元が
「動作検証中」と明記しているベータ品質のモジュールのため、**今回のスコープ外**。
別の設計ドキュメントで扱う)。

今回実装する4項目:

1. デバイス情報 → `cormoran/zmk-feature-device-info`
2. 安定性(ウォッチドッグ) → `cormoran/zmk-feature-watchdog`
3. キースイッチ → `cormoran/zmk-feature-kscan-diagnostics`
   (**注意**: DYA Studioの画面上の案内では必要モジュールが
   `cormoran/zmk-feature-watchdog` と表示されるが、これはDYA側
   (`cormoran/dya-studio`)の表示バグ。実際に`dya-studio`のソース
   (`src/components/troubleshooting/KscanDiagnosticsSection.tsx`)を
   確認したところ、要求している `MODULE_NAME` は
   `cormoran/zmk-feature-kscan-diagnostics` だった。こちらを実装する。)
4. スタック使用量 → `cormoran/zmk-module-devtool`(の `CONFIG_ZMK_DEVTOOL_STACK_USAGE`
   機能のみ。同モジュールが提供するキー注入・イベントタップ・ログキャプチャ等の
   他機能は今回のスコープ外)

## 互換性の確認(調査済み、再度やり直さないこと)

4モジュールとも、必要とする `zmk` のrevisionは `main+custom-studio-protocol`
であり、本リポジトリが現在使用中のものと**完全に一致する**。ブランチ切り替えは
不要。

## Owner decisions (do not re-litigate)

1. **`zmk-feature-watchdog` のsplit relay機能(`CONFIG_ZMK_WATCHDOG_SPLIT_RELAY`)
   は有効化しない。** 開発元のREADMEに「実機で繰り返しテストしたが、
   中継リクエストが最後まで完了したことがない(実験的機能)」と明記されている。
   ウォッチドッグ自体(`CONFIG_ZMK_WATCHDOG=y`)は左右両方の半分に対して個別に
   有効化し(各半分が自分のインシデントを独立して記録・保存する部分は
   実機検証済みで問題ない)、central側からは自分自身(右手)のログのみ閲覧できる
   状態にする。
2. **`zmk-feature-kscan-diagnostics` のオプション機能
   `zmk-feature-input-stream`(押下のリアルタイム可視化)は今回追加しない。**
   west.ymlへの追加なしで動作する(ポーリングによる統計表示にフォールバックする
   だけで、診断結果自体は同じ)ため、スコープを小さく保つ。
3. `zmk-feature-kscan-diagnostics` は左右分割対応(`CONFIG_ZMK_KSCAN_DIAGNOSTICS_SPLIT`)
   をそのまま使う(READMEにある通り、split構成では自動的に有効になる設定)。
   これにより左手側のキー配線・統計もStudio経由で見られるようになる
   (ただし開発元によると、peripheral側の配線情報を表示するWeb UI自体は
   まだ開発中とのことなので、現時点では「wiring info unavailable」と表示される
   場合がある。これはDYA側の実装状況次第であり、本リポジトリ側の対応範囲外)。
4. `zmk-module-devtool` は **`CONFIG_ZMK_DEVTOOL_STACK_USAGE` のみ**有効化する。
   同モジュールが提供するキー注入(`CONFIG_ZMK_DEVTOOL_KEY_INJECTION`)や
   イベントタップ(`CONFIG_ZMK_DEVTOOL_EVENT_TAP`、全キー入力を観測できてしまう)、
   ログキャプチャ(`CONFIG_ZMK_DEVTOOL_LOG_CAPTURE`)は、開発・テスト用途向けで
   今回の「スタック使用量を見たい」という要望には不要かつセキュリティ上の
   懸念があるため、**有効化しない**。レイヤー状態(`CONFIG_ZMK_DEVTOOL_LAYER_STATE`)
   はStudio RPC有効化時のデフォルトyのままで問題ない(無効化の手間をかけない)。
5. 各モジュールが推奨するStudio RPCバッファサイズは、既存の設定
   (`CONFIG_ZMK_STUDIO_RPC_RX_BUF_SIZE=128` は combo 機能導入時に追加済み)と
   重複しないよう、**新たに必要な値だけを追加**する。今回追加が必要なのは
   `CONFIG_ZMK_STUDIO_RPC_TX_BUF_SIZE=256`(kscan診断が推奨)のみ。
6. flash容量については、4モジュールをまとめて追加するため消費量の増加が
   見込まれる。CIビルド(`Build ZEN DYA firmware`)が成功することをもって
   確認する。万一 `zen_right_trackball_pmw3610_central` が容量不足になった場合は、
   実装を中断して `docs/design/` に状況を書き戻し、どの機能を見送るか改めて
   相談する(独断で機能を間引かない)。

## 実装手順(Codex 向け・最小限)

1. `config/west.yml` の `projects` に以下4件を追加する(`zmk` エントリの
   revisionは変更しない):
   ```yaml
   - name: zmk-feature-device-info
     remote: cormoran
     revision: main
   - name: zmk-feature-watchdog
     remote: cormoran
     revision: main+custom-studio-protocol
     import: true
   - name: zmk-feature-kscan-diagnostics
     remote: cormoran
     revision: main
     import: true
   - name: zmk-module-devtool
     remote: cormoran
     revision: main
     import: true
   ```

2. `boards/shields/zen/zen_right.conf` の末尾に以下を追加する:
   ```
   # Troubleshooting: デバイス情報
   CONFIG_ZMK_DEVICE_INFO=y
   CONFIG_ZMK_DEVICE_INFO_STUDIO_RPC=y

   # Troubleshooting: 安定性(ウォッチドッグ)
   CONFIG_ZMK_WATCHDOG=y
   CONFIG_ZMK_WATCHDOG_STUDIO_RPC=y

   # Troubleshooting: キースイッチ(kscan診断)
   CONFIG_ZMK_KSCAN_DIAGNOSTICS=y
   CONFIG_ZMK_KSCAN_DIAGNOSTICS_STUDIO_RPC=y
   CONFIG_ZMK_SPLIT_RELAY_EVENT_DATA_LEN=256
   CONFIG_ZMK_STUDIO_RPC_TX_BUF_SIZE=256

   # Troubleshooting: スタック使用量(devtool)
   CONFIG_ZMK_DEVTOOL=y
   CONFIG_ZMK_DEVTOOL_STUDIO_RPC=y
   CONFIG_ZMK_DEVTOOL_STACK_USAGE=y
   ```
   (`CONFIG_ZMK_STUDIO=y`、`CONFIG_ZMK_STUDIO_RPC_RX_BUF_SIZE=128`、
   `CONFIG_ZMK_LOW_PRIORITY_THREAD_STACK_SIZE=2048` は既存設定をそのまま使うため
   追加しない。)

3. `boards/shields/zen/zen_left.conf` の末尾に以下を追加する:
   ```
   # Troubleshooting: 安定性(ウォッチドッグ)— 左手分の記録用
   CONFIG_ZMK_WATCHDOG=y

   # Troubleshooting: キースイッチ(kscan診断)— 左手分のトポロジ・統計用
   CONFIG_ZMK_KSCAN_DIAGNOSTICS=y
   CONFIG_ZMK_SPLIT_RELAY_EVENT_DATA_LEN=256
   ```
   (左手側は `CONFIG_ZMK_STUDIO` を有効化しないため、`_STUDIO_RPC` 系の設定は
   追加しない。)

4. コミットしてpushする。ローカルビルド確認・GitHub Actionsの結果待ちは
   行わない(`codex-collaboration-workflow.md` のトークン節約ルール通り)。

5. 本ファイルの `Status:` 行を `implemented(CI成功待ち/ユーザーによる実機確認待ち)`
   に更新する。

## Files touched

- `config/west.yml`
- `boards/shields/zen/zen_right.conf`
- `boards/shields/zen/zen_left.conf`

`config/keymap.keymap`, `src/`, `proto/`, `web/` は変更不要
(DYA Studio側のWeb UIは既に各モジュールのRPCサブシステムに対応済みのため)。

## 次のステップ(このドキュメントの対象外)

実機での最終確認はユーザー(hrimfaxi-genius)が行う。ビルドしたUF2を両手に
書き込み、DYA Studioのトラブルシューティングタブで4項目(デバイス情報・
安定性・キースイッチ・スタック使用量)のエラーが解消し、それぞれ情報が
表示されることを確認する。

「トラックボールセンサー(PMW3610)」は本ドキュメントの対象外。動作検証中の
ドライバ置き換えが必要なため、別途設計ドキュメントで扱う。

## 実装後の追記(2026-09-04, Claude)

CIで2件の不具合が見つかり、Claudeが直接 `main` ブランチにコミットして修正した(Codexへの実装指示は不要だった)。

1点目: `config/west.yml` の `zmk-feature-watchdog` エントリの `revision` を、上の「実装手順」節および「互換性の確認」節に記載した `main+custom-studio-protocol` のまま設定したところ、Build ZEN DYA firmware #24 が `fatal error: couldn't find remote ref main+custom-studio-protocol` で失敗した。この特殊ブランチ名は `cormoran/zmk` フォーク自身(zmkエントリ)専用のものであり、`zmk-feature-watchdog` リポジトリ自体には存在しない(実際のブランチは `main` のみ)。他の3モジュール(device-info, kscan-diagnostics, devtool)は元から正しく `revision: main` だった。「互換性の確認」節の「4モジュールとも、必要とする `zmk` のrevisionは `main+custom-studio-protocol`」という記述も誤りで、正しくは「4モジュール自身は `revision: main` を使う(`main+custom-studio-protocol` は `zmk` フォーク自体のエントリにのみ適用される)」。コミット `ce9b02d` で修正。

2点目: 1点目の修正後、Build ZEN DYA firmware #24 は zen_left のビルドが `fatal error: proto/zmk/custom.pb.h: No such file or directory` で失敗した。原因は `zen_left.conf` にも `CONFIG_ZMK_STUDIO=y` が設定されている(split/lockingの都合で既存設定)ため、`CONFIG_ZMK_KSCAN_DIAGNOSTICS_STUDIO_RPC` がデフォルトでON(`depends on ZMK_STUDIO` かつ `default y`)になり、central専用のはずの `src/studio/*.c` がzen_left側でもビルド対象になってしまったこと。この際に必要な `proto/zmk/custom.pb.h` は `CONFIG_ZMK_STUDIO_RPC_CUSTOM_SUBSYSTEM_REQUEST_PAYLOAD_MAX_BYTES`(zen_right.confのみ設定)がある場合にのみ生成されるため、zen_leftでは見つからずビルドエラーになっていた。`zen_left.conf` に `CONFIG_ZMK_KSCAN_DIAGNOSTICS_STUDIO_RPC=n` を追加して解決(peripheral側はStudio RPCハンドラ不要で、`CONFIG_ZMK_KSCAN_DIAGNOSTICS_SPLIT`側だけで中継応答できる、というモジュール自身のKconfigヘルプ記載通り)。コミット `d3cd831`。

この2件の修正後、Build ZEN DYA firmware #25(commit d3cd831)・Test ZMK Module #24とも成功。CIは green。

Owner decision 1(watchdogのsplit relayを無効化する)について: `cormoran/zmk-feature-watchdog` のKconfigを確認したところ、`CONFIG_ZMK_WATCHDOG_SPLIT_RELAY` のヘルプ文が更新されており、「2026-07-07に実機検証済み。以前壊れていた原因は本モジュール内の残留デバッグコード(早期return)であり、トランスポートやタイミングの問題ではなかった」と明記されていた。これを受けてユーザーに確認したところ、最新情報を信じて有効のままにする方針に決定(このオプションは `default y if ZMK_SPLIT` のため、split構成である本リポジトリでは明示的な設定をしなくても既にONになっている。追加のconf変更は不要)。Owner decision 1はこの追記により更新版として扱う。

次のステップ: ユーザーが実機で4項目(デバイス情報・安定性・キースイッチ・スタック使用量)の動作確認を行う。Codex側の対応は、直接pushされた上記2コミット(`ce9b02d`, `d3cd831`)分の `git pull` による同期のみで、追加の実装作業は不要。
