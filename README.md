# Toshy Apple JIS キーボード向けフォーク

これは[Toshy](https://github.com/RedBearAK/Toshy)をフォークして、Apple 日本語（JIS）キーボードを Linux で使用するユーザー向けに、以下の修正・機能追加を行ったものです。

1. タブ切り替えショートカットの JIS 配列対応
2. 英数・かなキーによる IME 切り替え

> **Note**: 現在 Debian 13 + GNOME（X11 および Wayland セッション）でのみ動作確認しています。他のディストリビューション・デスクトップ環境では未検証です。

## セットアップ手順

### インストール

```bash
git clone https://github.com/miminashi/toshy.git
cd toshy
./setup_toshy.py install
```

### IME 設定

> **Note**: Wayland セッション（GNOME Wayland）では `ibus engine` コマンド方式を使用するため、以下の IBus ホットキー設定と Mozc キーマップ設定は**不要**です。X11 セッションでのみ必要です。

#### IBus のホットキー設定（X11 のみ）

IBus の `enable-unconditional` ホットキーを Katakana に設定する必要があります:

```bash
gsettings set org.freedesktop.ibus.general.hotkey enable-unconditional "['Katakana']"
```

#### Mozc のキーマップ設定（X11 のみ）

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

### 動作確認

セットアップ完了後、以下の手順で動作を確認します。

1. Toshy サービスが稼働していることを確認:

   ```bash
   systemctl --user status toshy-config toshy-session-monitor
   ```

2. キーリマッピングの動作確認:
   - **英数キー** を押す → IME がオフになる（直接入力）
   - **かなキー** を押す → ひらがな入力モードに切り替わる
   - **Cmd+Shift+\[** / **Cmd+Shift+\]** → 前のタブ / 次のタブに移動する

3. 問題がある場合はログを確認:

   ```bash
   journalctl --user -u toshy-config -f
   ```

## トラブルシューティング

### かなキーを押すたびに IME がトグルする

Toshy サービス（`toshy-config`）が停止している場合、かなキー（`KEY_HANGEUL`）が xwaykeyz で変換されずに IBus に直接到達します。IBus のデフォルト trigger リストには `Hangul` が含まれているため、トグル動作になります。

```bash
# サービスの状態を確認
systemctl --user status toshy-config

# サービスを再起動
systemctl --user restart toshy-config
```

### 英数・かなキーが反応しない

1. `toshy-config` サービスが稼働しているか確認:

   ```bash
   systemctl --user status toshy-config
   ```

2. **X11 セッションの場合**のみ、IBus の `enable-unconditional` 設定が正しいか確認:

   ```bash
   gsettings get org.freedesktop.ibus.general.hotkey enable-unconditional
   # 期待値: ['Katakana']
   ```

3. **X11 セッションの場合**のみ、Mozc のキーマップに無変換・変換キーの設定が入っているか確認（「[Mozc のキーマップ設定](#mozc-のキーマップ設定x11-のみ)」を参照）

4. **Wayland セッションの場合**、`ibus` コマンドが利用可能か確認:

   ```bash
   which ibus
   DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$(id -u)/bus ibus engine
   # Mozc エンジン名（mozc-jp, mozc-on, mozc-off のいずれか）が表示されること
   ```

## 技術解説

### タブ切り替えショートカットの JIS 配列対応

macOS の `Cmd+Shift+[` / `Cmd+Shift+]`（前のタブ / 次のタブ）に対応するキーバインドを、JIS 配列に合わせて修正しています。

- **前のタブ**: `Shift-RC-Left_Brace` → `Shift-RC-Right_Brace`
- **次のタブ**: `Shift-RC-Right_Brace` → `Shift-RC-Backslash`

US 配列では `[` と `]` はそれぞれ独立したキーですが、JIS 配列ではこれらのキーの物理位置が異なります。Linux の evdev では、JIS キーボードの `[` キーは `Right_Brace`（keycode 27）、`]` キーは `Backslash`（keycode 43）として認識されます。そのため、オリジナルの Toshy が使用する `Left_Brace`（keycode 26）ベースのバインドは JIS キーボードでは機能しません。

このフォークでは、上記 11 箇所のタブ切り替えバインドを JIS 配列の evdev キーコードに合わせて修正しています。

### 英数・かなキーによる IME 切り替え

Apple JIS キーボードの英数キー・かなキーで IME の ON/OFF を切り替えられるよう設定しています。

- **英数キー** → IME OFF（直接入力に切り替え）
- **かなキー** → IME ON（ひらがな入力に切り替え）

セッションタイプ（X11 / Wayland）に応じて異なる方式を使用します。

#### Wayland セッション（ibus engine 方式）

GNOME Shell の IBus 統合では、一部の keysym（`Henkan_Mode`、`Katakana`、`Zenkaku_Hankaku` 等）が IBus エンジンに転送されません。そのため、keysym ベースの IME 切り替えは使用できません。

代わりに `ibus engine` コマンドで Mozc のエンジンバリアントを直接切り替えます:

- **英数キー** → `ibus engine mozc-off`（DirectInput モードの Mozc に切り替え）
- **かなキー** → `ibus engine mozc-on`（ひらがなモードの Mozc に切り替え）

Mozc の IBus 設定には `mozc-on`（`composition_mode: HIRAGANA`）と `mozc-off`（`composition_mode: DIRECT`）のエンジンバリアントが定義されており、これらを切り替えることで確実にモード遷移できます。

> **Note**: この方式では IBus の `enable-unconditional` 設定や Mozc のキーマップ設定（変換/無変換キーのバインド）は不要です。

#### X11 セッション（二段キー方式）

X11 ではスタンドアロンの IBus デーモンがすべての keysym を正しくエンジンに転送するため、keysym ベースの方式が使用できます。

Apple JIS キーボードの英数・かなキーは、Linux の evdev では韓国語キーボード用のキーコードとして認識されます:
- 英数キー → `KEY_HANJA` (code 123)
- かなキー → `KEY_HANGEUL` (code 122)

英数キーは modmap で単純変換、かなキーは keymap で二段キーシーケンスを送信します:
- `KEY_HANJA` → `KEY_MUHENKAN` (code 94) — 無変換キー（modmap）
- `KEY_HANGEUL` → [`KEY_KATAKANA` (code 90), `KEY_HENKAN` (code 92)]（keymap）

この方式は 3 つの層で動作します:

**1. IBus エンジン制御（IBus 層）**

IBus の `enable-unconditional` が `KEY_KATAKANA` をトリガーとしてエンジンを再起動します。
IBus は KATAKANA キーイベントを消費するため、後続の HENKAN は常に Mozc に到達します。

- エンジンがアクティブな場合: KATAKANA は消費（no-op）→ HENKAN → Mozc
- エンジンが非アクティブな場合: KATAKANA でエンジン起動（消費）→ HENKAN → Mozc

> **Note**: IBus のデフォルト trigger リストには `Hangul`（= Apple JIS かなキーの evdev キーコード）が含まれています。xwaykeyz が正常に動作している場合、HANGEUL キーイベントは KATAKANA + HENKAN に変換されてから IBus に到達するため、この trigger は発火しません。Toshy サービスが停止している場合のみ、IBus が Hangul をトグルキーとして処理します。

**2. IME モード切替（Mozc 層）**

Mozc のキーマップ（`~/.config/mozc/config1.db`）で以下のコマンドが設定されている必要があります（設定手順は「[IME 設定](#ime-設定)」を参照）:
- 無変換(Muhenkan) → `IMEOff`（直接入力 — Mozc エンジン非アクティブ化）
- 変換(Henkan) → `InputModeHiragana`（ひらがなモードに切替）

---

オリジナルの Toshy README は [README.orig.md](README.orig.md) を参照してください。
