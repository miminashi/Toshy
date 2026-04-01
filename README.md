# Toshy Apple JIS キーボード向けフォーク

これは[Toshy](https://github.com/RedBearAK/Toshy)をフォークして、Apple 日本語（JIS）キーボードを Linux で使用するユーザー向けに、以下の修正・機能追加を行ったものです。

1. タブ切り替えショートカットの JIS 配列対応
2. 英数・かなキーによる IME 切り替え

> **Note**: 現在 Debian 13 + GNOME（X11 セッション）でのみ動作確認しています。Wayland セッション、および他のディストリビューション・デスクトップ環境では未検証です。

## セットアップ手順

### インストール

```bash
git clone https://github.com/miminashi/toshy.git
cd toshy
./setup_toshy.py install
```

### IME 設定

#### IBus のホットキー設定

IBus の `enable-unconditional` ホットキーを Katakana に設定する必要があります:

```bash
gsettings set org.freedesktop.ibus.general.hotkey enable-unconditional "['Katakana']"
```

#### Mozc のキーマップ設定

Mozc のキーマップに無変換・変換キーの動作を設定する必要があります:

1. Mozc の設定画面を開く（IBus パネルの Mozc アイコンを右クリック →「Properties」、または以下を実行）:

   ```bash
   /usr/lib/mozc/mozc_tool --mode=config_dialog
   ```

2. 「General」タブ →「Keymap style」の「Customize...」をクリック
3. キーマップエディタで以下のエントリを追加:

   | モード | 入力キー | コマンド |
   |---|---|---|
   | 入力文字なし | 無変換 | IMEを無効化 |
   | 変換前入力中 | 無変換 | IMEを無効化 |
   | 変換中 | 無変換 | IMEを無効化 |
   | 入力文字なし | 変換 | ひらがなに入力切替 |
   | 変換前入力中 | 変換 | ひらがなに入力切替 |
   | 変換中 | 変換 | ひらがなに入力切替 |

4. 「OK」で保存

### F1-F12 キー設定

Apple キーボードではトップ行のキーがデフォルトでメディアキー（輝度・音量等）として動作します。F1-F12 をデフォルトにするには、Toshy の `toshy-fnmode` ツールで `fnmode` を変更します（これは Toshy 本体の既存機能です）:

```bash
toshy-fnmode 2 --persistent
```

- `2`（fkeysfirst）: F1-F12 がデフォルト、Fn 押下でメディアキー
- 現在の設定を確認するには `toshy-fnmode --info` を実行

この設定は `/etc/modprobe.d/hid_apple.conf` に書き込まれ、再起動後も維持されます。

## 技術解説

### タブ切り替えショートカットの JIS 配列対応

macOS の `Cmd+Shift+[` / `Cmd+Shift+]`（前のタブ / 次のタブ）に対応するキーバインドを、JIS 配列に合わせて修正しています。

- **前のタブ**: `Shift-RC-Left_Brace` → `Shift-RC-Right_Brace`
- **次のタブ**: `Shift-RC-Right_Brace` → `Shift-RC-Backslash`

US 配列では `[` と `]` はそれぞれ独立したキーですが、JIS 配列ではこれらのキーの物理位置が異なります。Linux の evdev では、JIS キーボードの `[` キーは `Right_Brace`（keycode 27）、`]` キーは `Backslash`（keycode 43）として認識されます。そのため、オリジナルの Toshy が使用する `Left_Brace`（keycode 26）ベースのバインドは JIS キーボードでは機能しません。

このフォークでは、上記 11 箇所のタブ切り替えバインドを JIS 配列の evdev キーコードに合わせて修正しています。

### 英数・かなキーによる IME 切り替え（二段キー方式）

Apple JIS キーボードの英数キー・かなキーで IME の ON/OFF を切り替えられるよう設定しています。

- **英数キー** → IME OFF（直接入力に切り替え）
- **かなキー** → IME ON（ひらがな入力に切り替え）

この機能は 3 つの層で動作します:

**1. キーコードの変換（Toshy/xwaykeyz 層）**

Apple JIS キーボードの英数・かなキーは、Linux の evdev では韓国語キーボード用のキーコードとして認識されます:
- 英数キー → `KEY_HANJA` (code 123)
- かなキー → `KEY_HANGEUL` (code 122)

英数キーは modmap で単純変換、かなキーは keymap で二段キーシーケンスを送信します:
- `KEY_HANJA` → `KEY_MUHENKAN` (code 94) — 無変換キー（modmap）
- `KEY_HANGEUL` → [`KEY_KATAKANA` (code 90), `KEY_HENKAN` (code 92)]（keymap）

**2. IBus エンジン制御（IBus 層）**

IBus の `enable-unconditional` が `KEY_KATAKANA` をトリガーとしてエンジンを再起動します。
IBus は KATAKANA キーイベントを消費するため、後続の HENKAN は常に Mozc に到達します。

- エンジンがアクティブな場合: KATAKANA は消費（no-op）→ HENKAN → Mozc
- エンジンが非アクティブな場合: KATAKANA でエンジン起動（消費）→ HENKAN → Mozc

KATAKANA キーを選択した理由: Apple JIS キーボードに物理キーがなく、Mozc のキーマップにもバインドがないため安全。

> **Note**: IBus のデフォルト trigger リストには `Hangul`（= Apple JIS かなキーの evdev キーコード）が含まれています。xwaykeyz が正常に動作している場合、HANGEUL キーイベントは KATAKANA + HENKAN に変換されてから IBus に到達するため、この trigger は発火しません。Toshy サービスが停止している場合のみ、IBus が Hangul をトグルキーとして処理します。

**3. IME モード切替（Mozc 層）**

Mozc のキーマップ（`~/.config/mozc/config1.db`）で以下のコマンドが設定されている必要があります（設定手順は「[IME 設定](#ime-設定)」を参照）:
- 無変換(Muhenkan) → `IMEOff`（直接入力 — Mozc エンジン非アクティブ化）
- 変換(Henkan) → `InputModeHiragana`（ひらがなモードに切替）

英数キー側は `IMEOff` で真の直接入力になるため、半角英数モードのように変換候補 UI が表示されません。
かなキー側は二段キー方式により、エンジンが非アクティブでも KATAKANA → HENKAN の順で確実にひらがなモードに復帰します。

---

オリジナルの Toshy README は [README.orig.md](README.orig.md) を参照してください。
