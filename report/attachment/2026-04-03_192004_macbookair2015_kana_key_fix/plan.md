# MacBook Air 2015 11-inch かなキーでひらがなモードに切り替わらない問題の修正プラン

## Context

MacBook Air 2015 11-inch (Debian 13 / GNOME 48 / Wayland) で Toshy セットアップ後、「かな」キーを押してもひらがな入力モードに切り替わらない。

### 調査結果

**環境:**
- OS: Debian 13 (trixie), Kernel 6.12.74, GNOME Shell 48.7, Wayland
- Keyboard: Apple Inc. Apple Internal Keyboard / Trackpad (05AC:0290)
- IME: IBus + Mozc (mozc-jp)
- xwaykeyz: v1.16.1, Wayland mode (-w)
- 入力ソース: `[('ibus', 'mozc-jp'), ('xkb', 'jp')]`

**正常動作している部分:**
1. xwaykeyz が物理キーボードを正常に grab している
2. KBTYPE が "Apple" として検出される (デバイス名に "Apple" を含む)
3. `C("HANGEUL"): [Key.KATAKANA, Key.HENKAN]` keymap は正常にマッチする (テスト確認済み)
4. 仮想キーボードから KATAKANA + HENKAN が正しく出力される (テスト確認済み)
5. IBus `enable-unconditional: ['Katakana']` 設定済み
6. Mozc に `Precomposition Henkan InputModeHiragana` バインドあり

**根本原因:**

Mozc が DirectInput モード (IME OFF) の状態で `Henkan` キーを受け取っても、対応するキーバインドが存在しない。

Mozc の DirectInput モードのバインドは以下のみ:
```
DirectInput  Kanji   IMEOn
DirectInput  ON      IMEOn
```

`DirectInput Henkan` のバインドが**存在しない**。

GNOME Shell 48 (Wayland) の IBus 統合では、`enable-unconditional` (KATAKANA) トリガーは入力ソースの切り替え(xkb→ibus)のみ行い、**既に Mozc エンジンがアクティブな場合は何もしない**。X11 環境ではスタンドアロン IBus が `enable()` コールバックを再送信し、Mozc に `ON` イベントが届いて DirectInput から復帰していたが、GNOME Wayland では発生しない。

**結果のフロー (現在の動作):**
1. Mozc が DirectInput モード (英数キーで IME OFF にした後)
2. ユーザーが「かな」を押す → xwaykeyz が [KATAKANA, HENKAN] を出力
3. KATAKANA → GNOME: Mozc は既にアクティブ → **何もしない** (キーは消費される)
4. HENKAN → Mozc DirectInput モード → **バインドなし** → **何も起きない**

### 前回のレポートとの関係

`report/2026-04-01_235252_kana_key_toggle_fix.md` で修正した `Key.HANGEUL` → `C("HANGEUL")` の型不一致は解決済み。今回は xwaykeyz のキー変換は正常だが、IBus/Mozc 側の挙動が GNOME Wayland 環境で異なることが原因。

## 修正方針

Mozc の `config1.db` に `DirectInput Henkan InputModeHiragana` エントリを追加する。

これにより、Mozc がどのモードにあっても HENKAN キーでひらがなモードに切り替わる:
- DirectInput: `Henkan → InputModeHiragana` (新規追加)
- Precomposition: `Henkan → InputModeHiragana` (既存)
- Composition: `Henkan → InputModeHiragana` (既存)
- Conversion: `Henkan → InputModeHiragana` (既存)

## 実装手順

### Step 1: Mozc config1.db のバックアップ
```
ssh miminashi@macbookair2015.lan "cp ~/.config/mozc/config1.db ~/.config/mozc/config1.db.bak"
```

### Step 2: Mozc config1.db の修正

`~/.config/mozc/config1.db` は Protocol Buffers 形式。Field 42 (custom_keymap_table) に TSV 形式のキーマップテーブルが格納されている。

修正内容:
- `DirectInput\tON\tIMEOn` の後に `DirectInput\tHenkan\tInputModeHiragana` を追加
- protobuf の length varint を更新 (4838 → 4838 + 追加バイト数)

Python スクリプトで protobuf のバイナリを直接編集する:
1. config1.db を読み込み
2. Field 42 の keymap テーブル文字列を抽出
3. `DirectInput\tON\tIMEOn\n` の後に新エントリを挿入
4. Field 42 の length varint を更新
5. 書き戻し

### Step 3: Mozc サーバーの再起動
```
ssh miminashi@macbookair2015.lan "killall mozc_server"
```
Mozc サーバーは次回 IBus エンジン使用時に自動再起動される。

### Step 4: 動作検証

テスト用 uinput デバイスを使って以下のシーケンスを実行:
1. HANJA (英数) を送信 → Mozc DirectInput モードへ
2. HANGEUL (かな) を送信 → xwaykeyz が KATAKANA + HENKAN に変換
3. IBus/Mozc の状態を確認 → ひらがなモードになっていることを検証

検証には Python evdev + IBus GI バインディングを使用。

### 対象ファイル (リモート)
- `~/.config/mozc/config1.db` (Mozc キーマップ設定, protobuf 形式)

### 対象ファイル (ローカル: Toshy プロジェクト側の変更なし)
今回は Toshy のソースコード変更は不要。問題は Mozc のキーマップ設定に起因する。

## 検証方法

1. テスト用 Python スクリプトで uinput デバイスを作成し、キーイベントを注入
2. HANJA → HANGEUL のシーケンスで DirectInput → ひらがな切替を確認
3. HANGEUL を連続で押しても常にひらがなモードが維持されることを確認
4. 英数 → かな → 文字入力 のフローが正常であることを確認

## リスク

- config1.db の protobuf バイナリを直接編集するため、誤りがあると Mozc の設定が壊れる可能性
  → バックアップで対応。失敗時は `.bak` から復元
- Mozc のバージョンアップ時にキーマップがリセットされる可能性
  → セットアップスクリプトで自動設定する仕組みを将来的に検討
