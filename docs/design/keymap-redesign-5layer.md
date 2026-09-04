# Design: 新デザインのキーマップ(5レイヤー構成)を config/keymap.keymap に反映する

Status: design
Owner: nao / Claude(設計) / Codex(実装)

## 背景

ユーザーが新しくデザインしたキーマップ(添付ファイル `keymap.keymap`)を、
リポジトリの `config/keymap.keymap` に反映したい。

添付ファイルは、現行の `config/keymap.keymap`(GitHub `main` ブランチ、
2026-09-04 時点で確認)と比べてレイヤー構成がかなり異なる新デザイン:

- レイヤー数: 現行 7レイヤー(layer_0〜layer_6) → 新デザイン 5レイヤー
  (layer_0〜layer_4)。
- layer_0(ベース層): 左手ホームロー下段(Aキー)の左隣キーが
  `&mo 4` になっており、レイヤー4(システム/Fキー層)へ直接アクセスできる。
- layer_1: トラックボール/マウスレイヤー(現行と同内容、変更なし)。
- layer_2: 記号レイヤー(Symbols)。現行より記号を拡充。
- layer_3: ナビゲーション & テンキーレイヤー(現行の layer_3 とは内容が
  異なる新構成)。
- layer_4: システム/Fキーレイヤー(F1〜F12、USB/BLE出力切替、BT_SEL等)。
  レイヤー0の `&mo 4` と、レイヤー1(トラックボール層)内の `&mo 4`
  (スクロール発動用、`zen_right.overlay` の `pointing_listener` の
  `scroller { layers = <4>; }` が参照している層番号)の両方からアクセス
  される。この層番号が variable であることに変わりはないため、
  `zen_right.overlay` 側は変更不要。

**比較の結果、現行 `config/keymap.keymap` にのみ存在し、添付ファイルには
無い要素が2つあった:**

1. `&studio_unlock` キー(現行 layer_3 の先頭に配置)。
   `boards/shields/zen/zen_right.conf` で
   `CONFIG_ZMK_STUDIO_LOCKING=y` になっているため、このキーが無いと
   キーボード側から ZMK Studio のロックを解除できない
   (DYA Studio 連携の前提機能)。
2. `combos { bt_clear { ... } }` ブロック(記号レイヤー(layer_2)で
   物理キー位置 28・29 を同時押しすると `&bt BT_CLR` を発火するコンボ)。

ユーザーに確認した結果、**両方とも新デザインに復元した上で組み込む**方針
に決定した(2026-09-04)。

## Owner decisions (do not re-litigate)

1. `config/keymap.keymap` の内容全体を、下記「新しい keymap.keymap の
   内容」で完全に置き換える(部分編集ではなくファイル全体の置換)。
2. 上記の置き換え内容には、添付の新デザイン(layer_0〜layer_4)に加えて
   以下の2点を追加済み:
   - `/ { ... }` 直下、`keymap { ... }` の手前に、現行と同じ内容の
     `combos { bt_clear { ... } }` ブロックを追加(`key-positions = <28 29>;`
     `layers = <2>;` は物理レイアウト・layer_2 の位置とも変更が無いため
     現行のまま流用可能)。
   - `layer_4` の2段目(CAPSLOCK行)の先頭キー(添付ファイルでは
     `&trans`)を `&studio_unlock` に変更。
3. `boards/shields/zen/zen.keymap`・`zen_right.overlay`・`zen_left.overlay`・
   `zen_right.conf`・`zen_left.conf` など、他のファイルは変更不要
   (`zen.keymap` は `config/keymap.keymap` を include するだけ。
   `zen_right.overlay` の `scroller { layers = <4>; }` は新デザインでも
   layer_4 がそのまま存在するため参照が壊れない)。
4. コミットメッセージは変更内容がわかるもの
   (例: "Redesign keymap: 5-layer layout with restored studio_unlock and bt_clear combo")
   にする。

## 実装手順(Codex 向け・最小限)

1. `config/keymap.keymap` の内容を、以下のブロックの内容で完全に置き換える。

```
#include <behaviors.dtsi>
#include <dt-bindings/zmk/bt.h>
#include <dt-bindings/zmk/keys.h>
#include <dt-bindings/zmk/outputs.h>
#include <dt-bindings/zmk/pointing.h>

/ {
    combos {
        compatible = "zmk,combos";

        bt_clear {
            bindings = <&bt BT_CLR>;
            key-positions = <28 29>;
            layers = <2>;
        };
    };

    keymap {
        compatible = "zmk,keymap";

        layer_0 {
            // Base layer (変更点: 左手ホームロー下段 A の左隣キーを &mo 5 → &mo 4 に変更し、
            // レイヤー4(システム/Fキー層)へ直接アクセスできるようにした)
            bindings = <
&kp TAB    &kp Q       &kp W             &kp E     &kp R             &kp T  &kp Y      &kp U      &kp I  &kp O           &kp P           &kp BSPC
&mo 4      &kp A       &kp S             &kp D     &kp F             &kp G  &kp H      &kp J      &kp K  &kp L           &kp MINUS       &kp ENTER
&kp LSHFT  &kp Z       &kp X             &kp C     &kp V             &kp B  &kp SPACE  &kp SPACE  &kp N  &kp M           &kp COMMA       &kp DOT          &kp UP_ARROW  &mt RSHFT QUESTION
&mo 2      &kp ESCAPE  &kp LEFT_CONTROL  &kp LALT  &kp LEFT_COMMAND  &mo 2  &kp SPACE  &kp SPACE  &mo 3  &kp LEFT_ARROW  &kp DOWN_ARROW  &kp RIGHT_ARROW
            >;
        };

        layer_1 {
            // トラックボール/マウスレイヤー(変更なし・現状維持)
            bindings = <
&trans  &trans  &trans  &trans  &trans  &trans  &trans                &mkp LCLK  &mo 4             &mkp RCLK  &trans                 &trans
&trans  &trans  &trans  &trans  &trans  &trans  &kp RG(LEFT_BRACKET)  &mkp MB1   &mkp MB3          &mkp MB2   &kp RG(RIGHT_BRACKET)  &trans
&trans  &trans  &trans  &trans  &trans  &trans  &trans                &trans     &kp LC(UP_ARROW)  &trans     &trans                 &trans  &trans  &trans
&trans  &trans  &trans  &trans  &trans  &trans  &trans                &trans     &trans            &trans     &trans                 &trans
            >;
        };

        layer_2 {
            // 記号レイヤー(Symbols) - コーディングで頻出する記号を集約
            bindings = <
&kp ESCAPE  &kp N1     &kp N2  &kp N3     &kp N4         &kp N5          &kp N6         &kp N7             &kp N8     &kp N9                &kp N0                 &kp DELETE
&kp GRAVE   &kp EXCL   &kp AT  &kp HASH   &kp DOLLAR     &kp PRCNT       &kp CARET      &kp AMPERSAND      &kp ASTRK  &kp LEFT_PARENTHESIS  &kp RIGHT_PARENTHESIS  &kp PLUS
&trans      &kp MINUS  &kp UNDERSCORE  &kp EQUAL  &trans         &trans          &trans         &trans             &kp TILDE  &kp LEFT_BRACE        &kp RIGHT_BRACE        &kp LEFT_BRACKET  &kp RIGHT_BRACKET  &kp SLASH
&trans      &trans     &trans  &trans     &trans         &trans          &trans         &trans             &trans     &trans                &trans                 &trans
            >;
        };

        layer_3 {
            // ナビゲーション & テンキーレイヤー(Navigation & Numpad)
            // 矢印キーはベースレイヤー0に常設済みのため、ここでは Home/End/PageUp/PageDown とテンキーに専念
            bindings = <
&trans  &kp INSERT  &kp HOME  &kp PAGE_UP    &trans  &trans  &trans  &kp N7  &kp N8  &kp N9  &kp ASTRK  &kp BSPC
&trans  &kp DELETE  &kp END   &kp PAGE_DOWN  &trans  &trans  &trans  &kp N4  &kp N5  &kp N6  &kp PLUS   &kp ENTER
&trans  &trans      &trans    &trans         &trans  &trans  &trans  &trans  &trans  &kp N1  &kp N2     &kp N3  &trans  &kp SLASH
&trans  &trans      &trans    &trans         &trans  &trans  &trans  &trans  &trans  &kp N0  &kp DOT    &kp EQUAL
            >;
        };

        layer_4 {
            // システム/Fキーレイヤー(System / Function / Bluetooth)
            // レイヤー0の &mo 4(旧&mo5のキー)、およびレイヤー1(トラックボール層)の &mo 4 の両方からアクセス可能
            bindings = <
&kp F1        &kp F2        &kp F3        &kp F4        &kp F5        &kp F6      &kp F7      &kp F8  &kp F9  &kp F10  &kp F11  &kp F12
&studio_unlock  &kp CAPSLOCK  &trans        &trans        &trans        &trans      &trans      &trans  &trans  &trans   &trans   &trans
&out OUT_USB  &out OUT_BLE  &bt BT_NXT    &bt BT_CLR    &bt BT_CLR_ALL  &trans    &trans      &trans  &trans  &trans   &trans   &trans  &trans  &trans
&bt BT_SEL 0  &bt BT_SEL 1  &bt BT_SEL 2  &bt BT_SEL 3  &bt BT_SEL 4  &trans      &trans      &trans  &trans  &trans   &trans   &trans
            >;
        };
    };
};
```

2. コミットしてpushする。ローカルビルド確認・GitHub Actionsの結果待ちは
   行わない(`codex-collaboration-workflow.md` のトークン節約ルール通り)。
3. 本ファイルの `Status:` 行を `implemented` に更新する。

## Files touched

- `config/keymap.keymap`

