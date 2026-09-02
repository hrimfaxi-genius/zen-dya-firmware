# Design: 左右(central/peripheral)ペアリング不良の切り分け用デバッグログビルド

Status: implemented
Owner: hrimfaxi-genius / Claude(調査・設計) / Codex(実装)

## 背景

`docs/design/dya-ble-studio-visibility-fix.md`(Studio BLE可視化の修正、
commit `24d1ae1`)を実機に適用したところ、**別の問題**が見つかった。

- 右手(central, USB接続時は通常通りキー入力できる)
- 左手(peripheral)が右手に繋がらず、左手でのキー入力が一切反映されない

## これまでに確認済みの事実(切り分け済み、再度やり直さないこと)

1. 左右とも`settings_reset`でボンディング情報を一度クリアした上で、
   `zen_right_trackball_pmw3610_central.uf2` / `zen_left_peripheral.uf2`
   を書き込み直したが、症状は変わらない。
2. **`24d1ae1`(Studio ロック有効化)より前の、以前は左右とも正常に動作していた
   `44b63d0`時点のファームウェア**に戻して同様に settings_reset +
   書き込み直しを行っても、症状は変わらない(左右が繋がらない)。
   → **今回の Studio BLE 修正(`24d1ae1`)が原因ではないことがほぼ確定している。**
   この事実は重要なので、デバッグ対象のファームウェアは `24d1ae1` ベースのままでよい
   (`44b63d0`に戻す必要はない)。
3. 電池交換・電池接点の清掃を試したが、症状は変わらない(電源まわりの単純な
   接触不良ではなさそう)。
4. LED(`zmk-feature-status-led`)の観察結果:
   - 右→左の順で電源投入すると、両方とも起動時に3回点滅(バッテリー残量表示、
     71-100%を意味するので電池残量自体は問題ない)。
   - その後、**右は周期的な点滅を継続**(advertising中のパターンに見える =
     ずっと何かを探し続けている)。
   - **左は1回だけ点滅した後、完全に消灯したまま**沈黙する。
     (「接続確立時の短時間点灯」なのか、その直後に処理が停止/クラッシュ
     しているのかは点滅回数だけでは判別できない)

## 目的

LEDの点滅パターンだけでは判断できないので、**central(右手)をUSB接続した
状態で実際のブート/BLEログを見て**、以下を明らかにする。

- 右手(central)は左手(peripheral)を認識・スキャンできているか
- ペアリング/ボンディングの試行はどこまで進み、どこで失敗しているか
  (スキャンで見つからない/接続はするがGATT探索で失敗/認証エラー、など)
- 左手側が「1回点滅後に消灯」した直後に何が起きているか
  (正常に接続完了して省電力状態に入っただけなのか、パニック/リブートしているのか)

## Owner decisions (do not re-litigate)

1. **本番用の `zen_right.conf` / `zen_left.conf` / `build.yaml` の既存アーティファクト
   (`zen_right_trackball_pmw3610_central` / `zen_left_peripheral`)には手を加えない。**
   今回のログ取得は使い捨てのデバッグ用ビルドとして別アーティファクトを追加する形で行う。
   `docs/design/dya-ble-studio-visibility-fix.md` で直した内容を壊さないこと。
2. ベースにするのは現在の `main`(`24d1ae1`以降、Studio BLE修正込み)でよい。
   上記の通り今回の左右ペアリング不良はその修正とは無関係と切り分け済みのため。
3. ログの出力先は、**central(右手)に挿しているUSBケーブル経由**で見えるようにする
   ことを最優先とする(ユーザーは追加のデバッグプローブ(J-Link等)を持っていない
   前提)。現在central/peripheral共通で使っている `studio-rpc-usb-uart` snippet が
   USB CDC-ACMを専有しているため、ログ用のデバッグビルドではこのsnippetを
   一時的に外すか、共存可能な方法(例えばZMKやZephyrのUSBコンソール/RTT機能が
   同じCDC-ACMまたは別インターフェースを使えるか)をCodex側で確認して実装すること。
   正確なKconfig/デバイスツリー設定はZMK/Zephyrのドキュメントと、
   `dependencies/zmk`(west updateで取得される実体)のソースを見て確認すること
   (推測で書かない)。
4. ログレベルは、Bluetooth スタックおよび ZMK の split/central 関連モジュールで
   接続試行・スキャン・ボンディング・エラーが分かる程度(debug相当)まで上げる。
   一般的なキー入力等の大量ログは不要なので、モジュール単位で絞れるなら絞る。
5. このデバッグビルドは `build.yaml` に新しいアーティファクト
   (例: `zen_right_trackball_pmw3610_central_debug`)として追加する形にし、
   既存の2アーティファクトの定義はコピーしてsnippet/configだけ変える。
   目的を終えたら(ユーザーがログを取得し終えたら)この一時アーティファクトは
   削除して良い、という前提で作る。

## 実装手順(Codex 向け)

1. `dependencies/zmk`(または`west update`後に取得される実体)のソースを確認し、
   USB経由でZMK/Zephyrのログコンソールを見る標準的な方法(該当するKconfig名、
   `studio-rpc-usb-uart` snippetとの共存可否)を調べる。共存できない場合は、
   デバッグビルド専用の snippet 構成にする(例えば `studio-rpc-usb-uart` を外し、
   代わりにUSBコンソールログ用の設定を入れる)。
2. Bluetooth/splitまわりのログレベルを上げる設定を追加する
   (具体的なConfig名は上記調査結果に基づいて正しいものを使うこと)。
3. `build.yaml` に新しいデバッグ用アーティファクト
   (central側のみでよい。peripheral側の内部状態はcentral側のログから
   ある程度推測できるはずだが、必要なら peripheral 用のデバッグ artifact も
   同様に追加してよい)を追加する。
4. ローカルの `python -m unittest` でビルドが通ることを確認する
   (Renodeテストは新規追加した一時アーティファクトに対しては必須ではない)。
5. コミットし、GitHub Actions のビルドが成功することを確認する。
6. ユーザー向けに、README や本設計ドキュメントの追記として、
   「このデバッグ用UF2を central(右手)に書き込み、シリアルターミナル
   (例: PuTTY, Tera Term, `west espressif monitor`相当のもの、あるいは
   単純なUSBシリアルモニタ)で該当のCOMポートを開き、ボーレートいくつで
   接続すればログが見えるか」を明記する。
7. **実機でのログ採取自体はユーザーが行う。** 右手をUSB接続してシリアル
   ターミナルを開いた状態で、(a)右手電源投入直後、(b)続けて左手電源投入、
   の一連のログをコピーしてユーザーから共有してもらい、そのログをもとに
   Claude側で原因を特定する。

## Files touched(想定)

- `build.yaml`(デバッグ用アーティファクトの追加)
- `boards/shields/zen/zen_right_debug.conf` や `.overlay` など、
  デバッグ専用の設定ファイル(新規。既存の `zen_right.conf` は変更しない)
- 必要に応じて `docs/design/split-pairing-debug-logging.md`(本ファイル)に
  「ログの見方」を追記

## 実装したログ構成と採取手順

デバッグ用アーティファクトは
`zen_right_trackball_pmw3610_central_debug.uf2`。通常版とは異なり、
`studio-rpc-usb-uart` を外してZMK公式の `zmk-usb-logging` snippetを使用する。
このため、デバッグ版のUSB CDC-ACMポートはStudio RPCではなくログコンソール
専用となる。本番用のcentral/peripheralアーティファクトは変更していない。

Windowsでの採取手順:

1. デバッグ用UF2を右手(central)だけに書き込む。左手は通常の
   `zen_left_peripheral.uf2` のままでよい。
2. 右手をUSB接続し、デバイスマネージャーの「ポート (COMとLPT)」で新しく
   現れたCOMポートを確認する。
3. PuTTYやTera Termで、そのCOMポートを **115200 baud / 8 data bits / no
   parity / 1 stop bit / flow controlなし** で開く。CDC-ACMでは実際の転送速度は
   USB側で決まるが、端末設定はZephyrの既定line codingである115200 8N1に合わせる。
4. ターミナルを開いたまま右手を再起動し、ブートログが始まるのを待つ。
   Windowsでポートを開く猶予を確保するため、ログ処理開始を8秒遅延させている。
5. 右手のブートログを採取後、左手の電源を入れ、接続または失敗が落ち着くまで
   続けてログを採取する。
6. 右手起動直後から左手投入後までを省略せずテキストとして共有する。特に
   `Scanning`, `[DEVICE]`, `Connected`, `Security`, `Discover`, `SMP`, `ATT`,
   `GATT`, `Disconnected`, `err`, `reason` を含む行が切り分けに重要。

ログ取得後は電池消費とログ用メモリ負荷を避けるため、右手を通常版の
`zen_right_trackball_pmw3610_central.uf2` に戻すこと。
