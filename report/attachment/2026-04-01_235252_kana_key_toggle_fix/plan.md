# かなキー トグル問題の修正プラン

## Context

upstreamマージ後の再インストールにより、かなキーを押すたびにIME入力モードがトグルするようになった。
原因を調査し、keymap のキー指定方法のバグを発見した。

## 原因分析

`default-toshy-config/toshy_config.py` の keymap エントリで、辞書キーに **bare `Key` enum** (`Key.HANGEUL`) を使用している:

```python
keymap("Cond keymap - Apple JIS - Kana", {
    Key.HANGEUL:  [Key.KATAKANA, Key.HENKAN],   # ← ここが問題
}, when = ...)
```

xwaykeyz の `transform_key()`（`transform.py:923`）は keymap のルックアップ時に `Combo(get_pressed_mods(), key)` を生成して辞書検索する。

- `Key.HANGEUL` は `IntEnum` → `hash(122)`
- `Combo([], Key.HANGEUL)` → `hash((frozenset(), Key.HANGEUL))` — 異なるハッシュ

このため **このマッピングは一度もマッチしていない**。

結果のフロー:
1. HANGEUL キーが xwaykeyz の keymap を素通り
2. IBus のデフォルト trigger リストに `Hangul` が含まれている（`ibus read-config` で確認）
3. IBus が Hangul をトグルキーとして処理 → 毎回入力モードがトグル

## 修正内容

### ファイル1: `default-toshy-config/toshy_config.py`（テンプレート）

`Key.HANGEUL` を `C("HANGEUL")` に変更:

```python
keymap("Cond keymap - Apple JIS - Kana", {
    C("HANGEUL"):  [Key.KATAKANA, Key.HENKAN],
}, when = ...)
```

### ファイル2: `~/.config/toshy/toshy_config.py`（インストール済み設定）

同じ変更を適用。

出力側の `[Key.KATAKANA, Key.HENKAN]` は `handle_commands()`（`transform.py:1096`）が `isinstance(command, Key)` で `_output.send_key()` を呼び出すため、変更不要。

## 検証

1. toshy-config サービスを再起動: `systemctl --user restart toshy-config`
2. かなキーを押して、常にひらがなモードに切り替わることを確認（トグルしないこと）
3. 英数キーを押して、IME がオフになることを確認

## レポート作成

修正後、以下のレポートを作成する:

- ファイル名: `report/2026-04-01_234649_kana_key_toggle_fix.md`
- タイトル: 「かなキー IME トグル問題の調査・修正レポート」
- 内容:
  - 前提・目的（upstream マージ後に発生した問題）
  - 環境情報（OS, DE, xwaykeyz version, キーボード等）
  - 原因分析（keymap の Key vs Combo ハッシュ不一致）
  - 修正内容
  - 検証結果
- 添付ファイル:
  - `report/attachment/2026-04-01_234649_kana_key_toggle_fix/plan.md` にプランファイルをコピー
