# かなキー IME トグル問題の調査・修正レポート

- **実施日時**: 2026年4月1日 23:52 (JST)

## 添付ファイル

- [実装プラン](attachment/2026-04-01_235252_kana_key_toggle_fix/plan.md)

## 前提・目的

upstream (`RedBearAK/toshy`) をマージして `./setup_toshy.py install` で再インストールしたところ、Apple JIS キーボードの「かな」キーを押すたびに IME 入力モードがトグルするようになった。期待動作は「かなキーで常にひらがなモードへ切り替え（トグルではない）」。

- 背景: commit `549844d` で upstream/main をマージ。upstream の変更は XKB チェック機能の追加のみで、かなキー関連の変更なし
- 目的: トグル動作の原因を特定し修正する

## 環境情報

- OS: Debian 13 (trixie), Linux 6.12.74+deb13+1-amd64
- DE: GNOME (X11 セッション)
- Python: 3.13
- キーボード: Apple Inc. Apple Keyboard (USB, 05AC:021F)
- Toshy: main ブランチ (config version 20260330)
- xwaykeyz: v1.16.1
- IME: IBus + Mozc

## 原因分析

### keymap の辞書キーとルックアップの型不一致

`default-toshy-config/toshy_config.py` の keymap エントリで、辞書キーに bare `Key` enum を使用していた:

```python
keymap("Cond keymap - Apple JIS - Kana", {
    Key.HANGEUL:  [Key.KATAKANA, Key.HENKAN],   # bare Key enum
}, ...)
```

xwaykeyz の `transform_key()` (`transform.py:923`) は keymap ルックアップ時に `Combo(get_pressed_mods(), key)` を生成して辞書検索する:

```python
combo = Combo(get_pressed_mods(), key)  # Combo オブジェクト
for keymap in _active_keymaps:
    if combo not in keymap:     # dict の __contains__ で検索
        continue
```

Python の dict lookup はハッシュ値で一致判定するが:

- `Key.HANGEUL` は `IntEnum` → `hash(122)`
- `Combo([], Key.HANGEUL)` → `hash((frozenset(), Key.HANGEUL))` — **異なるハッシュ**

このため、**このマッピングは一度もマッチしていなかった**。

### 結果として起きていたこと

1. かなキー (HANGEUL) が xwaykeyz の keymap を素通り
2. IBus のデフォルト trigger リストに `Hangul` が含まれている（`ibus read-config` で確認: `trigger: ['Control+space', 'Zenkaku_Hankaku', 'Alt+Kanji', 'Alt+grave', 'Hangul', 'Alt+Release+Alt_R']`）
3. IBus が Hangul をトグルキーとして処理 → 毎回入力モードがトグル

なお、英数キー（HANJA）は modmap で `Key.HANJA: Key.MUHENKAN` と定義されており、modmap は bare Key での 1:1 置換のため正常に動作していた。

## 修正内容

`Key.HANGEUL` を `C("HANGEUL")` に変更し、正しい `Combo` オブジェクトを keymap の辞書キーとして使用するようにした。

### 変更箇所

- `default-toshy-config/toshy_config.py` (テンプレート)
- `~/.config/toshy/toshy_config.py` (インストール済み設定)

```diff
 keymap("Cond keymap - Apple JIS - Kana", {
-    Key.HANGEUL:  [Key.KATAKANA, Key.HENKAN],
+    C("HANGEUL"):  [Key.KATAKANA, Key.HENKAN],
 }, when = lambda ctx:
```

出力側の `[Key.KATAKANA, Key.HENKAN]` は `handle_commands()` が `isinstance(command, Key)` で `_output.send_key()` を呼び出すため、変更不要。

## 再現方法

1. `default-toshy-config/toshy_config.py` の keymap で `C("HANGEUL")` を `Key.HANGEUL` に戻す
2. `~/.config/toshy/toshy_config.py` にも同じ変更を反映
3. `systemctl --user restart toshy-config`
4. かなキーを複数回押す → 入力モードがトグルする（ひらがな→直接入力→ひらがな→...）

## 検証結果

1. `systemctl --user restart toshy-config` で再起動 → エラーなし、Apple Keyboard を正常に grab
2. かなキーを押す → 常にひらがなモードに切り替わることを確認（トグルしない）
3. 英数キーを押す → IME がオフになることを確認
