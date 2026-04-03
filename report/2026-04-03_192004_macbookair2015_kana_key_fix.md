# MacBook Air 2015 11-inch かなキーでひらがなモードに切り替わらない問題の調査・修正レポート

- **実施日時**: 2026年4月3日 19:20 〜 22:33 (JST)

## 添付ファイル

- [実装プラン](attachment/2026-04-03_192004_macbookair2015_kana_key_fix/plan.md)

## 前提・目的

MacBook Air 2015 11-inch (macbookair2015.lan) に Toshy をセットアップしたが、「かな」キーを押してもひらがな入力モードに切り替わらない問題の原因を特定し修正する。

- 背景: commit `ba10ede` で適用した `C("HANGEUL")` 修正（前回レポート参照）は反映済み。xwaykeyz のキー変換自体は正常に動作しているが、IME が反応しない
- 参照: [かなキー IME トグル問題の調査・修正レポート](2026-04-01_235252_kana_key_toggle_fix.md) — 前回の修正（X11 環境）
- 目的: GNOME Wayland 環境固有の原因を特定し修正する

## 環境情報

- OS: Debian 13 (trixie), Linux 6.12.74+deb13+1-amd64
- DE: GNOME Shell 48.7 (Wayland セッション)
- Python: 3.13
- キーボード: Apple Inc. Apple Internal Keyboard / Trackpad (USB, 05AC:0290) — MacBook Air 2015 内蔵 JIS キーボード
- Toshy: main ブランチ (config version 20260330)
- xwaykeyz: v1.16.1 (Wayland mode, -w フラグ)
- IME: IBus + Mozc (ibus-mozc 2.29.5160.102+dfsg-1.4)
- GNOME 入力ソース: `[('ibus', 'mozc-jp'), ('xkb', 'jp')]`

## 調査の過程

### Phase 1: 初期仮説の検証と棄却（Mozc DirectInput 問題）

最初に、Mozc の DirectInput モードに `Henkan` キーバインドが存在しないことを原因と仮定した。

**調査内容:**

1. xwaykeyz の仮想キーボード出力を監視するスクリプトを作成し、テスト用 uinput デバイスから KEY_HANGEUL を注入。xwaykeyz の keymap が `C("HANGEUL"): [Key.KATAKANA, Key.HENKAN]` を正しくマッチし、仮想キーボードから KATAKANA + HENKAN が出力されることを確認
2. Mozc の `config1.db`（protobuf 形式）をバイナリ解析し、Field 42 (custom_keymap_table) に DirectInput モードのバインドが `Kanji → IMEOn` と `ON → IMEOn` のみで、`Henkan` がないことを発見
3. GNOME Shell 48 (Wayland) の IBus 統合では、`enable-unconditional` (KATAKANA) が既にアクティブなエンジンに対して何もしないため、Mozc が DirectInput のまま HENKAN を受け取るが、バインドがなく無視されるという仮説を立てた

**修正試行:**

- `config1.db` に `DirectInput\tHenkan\tInputModeHiragana` を追加（protobuf の length varint も更新）
- `session_keymap` (Field 41) を NONE (0) → CUSTOM (1) に変更（カスタムキーマップテーブルが使用されるよう切り替え）
- mozc_server を再起動

**結果: 効果なし。** ユーザーの物理テストでかなキーは依然として無反応。

### Phase 2: キー配信経路の切り分け

Mozc 設定の問題ではなく、キーイベントが Mozc に到達していない可能性を検証するため、キー配信パイプラインの各段階を個別にテストした。

**Step 1: 物理キーイベントのキャプチャ**

xwaykeyz の仮想キーボードを 60 秒間監視するバックグラウンドロガーを起動し、ユーザーに MacBook Air 本体で英数・かなキーを押してもらった。

```
[19:27:22] KEY_MUHENKAN (code=94) DOWN/UP   ← 英数キー（正常に変換）
[19:27:23] KEY_KATAKANA (code=90) DOWN/UP   ← かなキー（正常に変換）
[19:27:23] KEY_HENKAN (code=92) DOWN/UP
```

→ xwaykeyz のキー変換は物理キーでも正常に動作。

**Step 2: libinput レベルの確認**

`sudo libinput debug-events --device /dev/input/event5` で仮想キーボードのイベントを確認:

```
event5   KEYBOARD_KEY   KEY_HENKAN (92) pressed
event5   KEYBOARD_KEY   KEY_HENKAN (92) released
```

→ libinput は HENKAN イベントを正常に処理。

**Step 3: Wayland コンポジター（Mutter）の確認**

`wev -f wl_keyboard`（Wayland Event Viewer）をユーザーに MacBook Air 本体で実行してもらい、キーイベントを記録:

```
key: 102; sym: Muhenkan     (65314)   ← 英数キー → Wayland クライアントに配信 ✓
key: 100; sym: Henkan_Mode  (65315)   ← かなキー → Wayland クライアントに配信 ✓
```

→ Mutter は HENKAN を `Henkan_Mode` keysym に正しく変換し、Wayland クライアントに配信している。問題は Mutter より後段（GNOME Shell IBus 統合 → Mozc）にある。

### Phase 3: keysym 単位の動作テスト（決定打）

**問題の分離:**

MUHENKAN は動作するが HENKAN は動作しない。両者の唯一の違いは keysym（keycode）自体。配信方式（modmap / keymap）の差ではないことを確認するため、以下のテストを実施した:

1. **かなキー → MUHENKAN (modmap)**: xwaykeyz の modmap で `Key.HANGEUL: Key.MUHENKAN` に設定し、かなキーを押すと MUHENKAN が送信されるようにした

   → ユーザーが Super+Space でひらがなモードにした後、かなキーを押すと **IME OFF になった**（動作する）

2. **かなキー → HENKAN (modmap)**: 同じ modmap で `Key.HANGEUL: Key.HENKAN` に変更

   → かなキーを押しても **何も起きない**（動作しない）

3. **かなキー → ZENKAKUHANKAKU (keymap)**: keymap で `C("HANGEUL"): Key.ZENKAKUHANKAKU` に設定

   → かなキーを押しても **何も起きない**（動作しない）

**結論:**

同じ modmap、同じ仮想キーボード、同じ配信経路で、**MUHENKAN (0xFF22) だけが IBus エンジンに転送され、HENKAN (0xFF23)、KATAKANA (0xFF26)、ZENKAKUHANKAKU (0xFF2A) は転送されない**。これは GNOME Shell 48 の IBus 統合における keysym 転送の制限であると判断した。

### Phase 4: 代替方式の探索

keysym ベースの方式が使えないため、以下の代替手段を試行した:

1. **`ibus engine mozc-on` コマンド**: Mozc の IBus 設定には `mozc-on`（`composition_mode: HIRAGANA`）と `mozc-off`（`composition_mode: DIRECT`）のエンジンバリアントが定義されている。xwaykeyz の keymap で callable を使い、かなキー押下時に `subprocess.Popen(["ibus", "engine", "mozc-on"])` を実行

   → **ひらがな入力に切り替わった。**

2. 英数キーも同じ方式（`ibus engine mozc-off`）に変更

   → **英数・かなの繰り返しで安定動作。**

## 原因分析（まとめ）

### GNOME Shell IBus 統合の keysym 転送制限（根本原因）

GNOME Shell 48 の IBus 統合において、特定の keysym が IBus エンジン (Mozc) に転送されない。

| keysym | evdev keycode | Wayland クライアント配信 | IBus エンジン転送 | 結果 |
|--------|--------------|----------------------|----------------|------|
| Muhenkan (0xFF22) | 94 | ✓ | ✓ | IMEOff 正常動作 |
| Henkan_Mode (0xFF23) | 92 | ✓ | ✗ | 無反応 |
| Katakana (0xFF26) | 90 | ✓ | ✗ | 無反応 |
| Zenkaku_Hankaku (0xFF2A) | 85 | ✓ | ✗ | 無反応 |

Wayland コンポジター (Mutter) → Wayland クライアントへの keysym 配信は正常だが、GNOME Shell の IBus 統合レイヤーがこれらの keysym を IBus エンジンに転送しない。Muhenkan のみが例外的に転送される。

### X11 環境では動作する理由

X11 ではスタンドアロンの IBus デーモンがキーイベントを処理するため、すべての keysym が正しくエンジンに転送される。GNOME Wayland では GNOME Shell が IBus を内部統合しており、keysym 転送に上記の制限がある。

### 初期仮説（Mozc DirectInput 問題）が誤りだった理由

当初は Mozc のキーマップに `DirectInput Henkan` バインドがないことが原因と考えた。しかし、HENKAN keysym 自体が Mozc に転送されないため、キーマップの有無は無関係だった。`config1.db` の修正と `session_keymap` の変更は効果がなかった。

## 修正内容

### 最終的な修正方針

`SESSION_TYPE` に応じた分岐を実装:

- **Wayland**: `ibus engine mozc-on/mozc-off` コマンドで Mozc エンジンバリアントを直接切り替え
- **X11**: 従来の keysym 方式（MUHENKAN modmap + KATAKANA+HENKAN keymap）を維持

### 変更箇所

**`default-toshy-config/toshy_config.py`** (テンプレート):

```python
if SESSION_TYPE == 'wayland':
    # Wayland: ibus engine コマンド方式
    _ibus_env = {**os.environ, "DBUS_SESSION_BUS_ADDRESS":
                 f"unix:path=/run/user/{os.getuid()}/bus"}

    keymap("Cond keymap - Apple JIS - Eisu", {
        C("HANJA"):  [
            lambda: subprocess.Popen(
                ["ibus", "engine", "mozc-off"],
                stdout=DEVNULL, stderr=DEVNULL, env=_ibus_env,
            ),
        ],
    }, when = ...)

    keymap("Cond keymap - Apple JIS - Kana", {
        C("HANGEUL"):  [
            lambda: subprocess.Popen(
                ["ibus", "engine", "mozc-on"],
                stdout=DEVNULL, stderr=DEVNULL, env=_ibus_env,
            ),
        ],
    }, when = ...)

else:
    # X11: keysym 方式（従来どおり）
    modmap("Cond modmap - Apple JIS - Eisu", {
        Key.HANJA:    Key.MUHENKAN,
    }, when = ...)

    keymap("Cond keymap - Apple JIS - Kana", {
        C("HANGEUL"):  [Key.KATAKANA, Key.HENKAN],
    }, when = ...)
```

**`~/.config/toshy/toshy_config.py`** (macbookair2015.lan インストール済み設定):
- 同様の変更を適用済み

**`README.md`**:
- 動作確認環境を「X11 および Wayland」に更新
- IME 設定セクションに「X11 のみ」の注記を追加
- 技術解説を Wayland / X11 の両方式に対応するよう書き直し

### Mozc 設定の変更（最終的に不要と判定）

当初 `config1.db` に `DirectInput Henkan InputModeHiragana` を追加し、`session_keymap` を CUSTOM (1) に変更したが、HENKAN keysym 自体が Mozc に転送されないことが判明したため、これらの変更は効果がなかった。残しても害はないが、修正の本質ではない。

## 再現方法

1. GNOME Wayland (GNOME Shell 48) 環境で Toshy をセットアップ
2. Apple JIS キーボードの英数・かなキーで keysym ベースの IME 切り替えを試みる
3. かなキー → HENKAN/KATAKANA keysym が Mozc に転送されない → ひらがなモードに切り替わらない

## 検証結果

### 物理キーテスト（macbookair2015.lan）

- [x] かなキーを押す → ひらがなモードに切り替わる
- [x] 英数キーを押す → 直接入力モードに切り替わる
- [x] 英数 → かな → 英数 → かな の繰り返しで安定して切り替わる（トグルではなく、常に期待するモードへ切り替わる）

## 今後の課題

- `subprocess.Popen` によるプロセス生成のオーバーヘッド（体感では問題なし）
- GNOME Shell の IBus 統合で特定の keysym が転送されない問題について、GNOME upstream への報告を検討
- IBus 以外の IME フレームワーク（Fcitx5 等）での動作は未検証
