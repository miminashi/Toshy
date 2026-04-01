# CLAUDE.md — Toshy プロジェクトガイド

## プロジェクト概要

Toshy は Linux（X11 + Wayland）向けの Mac スタイルキーボードショートカットリマッパー。
Python 3.8+ で動作し、コンパイル不要。キーマッパーエンジンとして xwaykeyz（keyszer のフォーク、`keymapper-temp/` サブプロジェクト）を使用する。

## 開発コマンド

### インストール
```bash
./setup_toshy.py install
# 主なフラグ: --barebones-config, --skip-native, --dev-keymapper
```

### テスト（xwaykeyz のみ、メイン Toshy にはテストスイートなし）
```bash
cd keymapper-temp && pytest .              # 全テスト
cd keymapper-temp && pytest tests/test_basics.py  # 単体テスト
```

### Lint チェック（xwaykeyz）
```bash
cd keymapper-temp && bash tools/check.sh
# codespell, bandit, flake8, black, isort, pytest, safety を実行
```

メイン Toshy プロジェクトにはトップレベルの Makefile やテストスイートはない。

## アーキテクチャ

### ディレクトリ構成
- `toshy_common/` — 共通インフラ（環境検出、SQLite 設定管理、サービス/プロセス管理、モニタリング）
- `toshy_gui/` — 設定 GUI（GTK-4 メイン、Tkinter フォールバック）
- `toshy_tray.py` — システムトレイインジケータ
- `default-toshy-config/` — キーマッパー設定テンプレート（フル版とベアボーン版）
- `keymapper-temp/` — xwaykeyz サブプロジェクト（hatchling ビルドシステム）
- `scripts/` — インストール・サービス管理シェルスクリプト
- `systemd-user-service-units/` — systemd サービス定義
- `cosmic-dbus-service/`, `kwin-dbus-service/`, `wlroots-dbus-service/` — D-Bus サービス
- `setup_toshy.py` — メインインストーラー（約8000行以上）、ネイティブパッケージ + pip venv セットアップ

### 主要パターン
- **環境検出**: `EnvironmentInfo`（`toshy_common/env_context.py`）が実行時にディストロ、DE、セッションタイプ、WM を自動検出
- **設定管理**: SQLite で永続化（`~/.config/toshy/toshy_user_preferences.sqlite`）
- **バックグラウンド動作**: systemd ユーザーサービス（toshy-config, toshy-session-monitor）
- **キーリマッピング定義**: 設定ファイル（`toshy_config.py`）が xwaykeyz の `config_api` を使用

## レポート作成ルール

- レポートはプロジェクトルート以下の `report` ディレクトリに作成する
- レポートのタイトルは日本語で記載する
- レポートには日時（分まで）を入れる
- レポートのファイル名は `yyyy-mm-dd_hhmmss_レポート名.md` にする（ファイル名のレポート名は英語）
- タイムスタンプは `TZ=Asia/Tokyo date +%Y-%m-%d_%H%M%S` コマンドで取得すること（LLM が時刻を推測してはならない）
- レポート内の日時表記は JST (日本標準時) で記載すること。システムが UTC の場合は +9 時間に変換する
- 実験やタスクの前提条件・目的は専用のセクションを設けて記載する
- 実験の再現方法（手順・コマンド等）を記載する
- 実験に際して参照した過去のレポートがある場合は、そのレポートへのリンクを記載する
- 実験レポートにはサーバ構成・ストレージ構成等の環境情報を記載する
- レポートに添付ファイル（プランファイル、ログ、スクリーンショット等）がある場合は `report/attachment/<レポートファイル名>/` ディレクトリに格納し、レポート本文から相対パスでリンクすること
  - `<レポートファイル名>` は `.md` を除いたファイル名（例: `2026-02-21_143052_ceph_cluster_setup`）
  - リンク例: `[実装プラン](attachment/2026-02-21_143052_ceph_cluster_setup/plan.md)`
- **プランファイルの添付（必須）**: プランモードで作業を行った場合、レポート作成時に必ず以下の手順でプランファイルを添付すること:
  1. 添付ディレクトリを作成: `mkdir -p report/attachment/<レポートファイル名>/`
  2. プランファイルをコピー: `cp /home/ubuntu/.claude/plans/<plan-name>.md report/attachment/<レポートファイル名>/plan.md`
     - `<plan-name>` はプランモード開始時に指定されたファイル名（例: `groovy-humming-candy`）
  3. レポート本文に `## 添付ファイル` セクションを設け、リンクを記載:
     ```markdown
     ## 添付ファイル

     - [実装プラン](attachment/<レポートファイル名>/plan.md)
     ```

### Discord 通知

レポート作成時（Write ツールで `report/` 直下に `.md` を書き込んだ時）、PostToolUse hook により Discord webhook で自動通知される。Webhook URL は `.env` の `DISCORD_WEBHOOK_URL` で設定する。

### 例

```
report/
  2026-03-15_102030_wayland_gnome_keymap_test.md
  attachment/
    2026-03-15_102030_wayland_gnome_keymap_test/
      plan.md
```

ファイル内の例:
````markdown
# Debian 13 Wayland + GNOME 環境での JIS キーボード動作検証レポート

- **実施日時**: 2026年3月15日 10:20

## 添付ファイル

- [実装プラン](attachment/2026-03-15_102030_wayland_gnome_keymap_test/plan.md)

## 前提・目的

Debian 13 の Wayland + GNOME 環境で Toshy の JIS キーボードリマッピングが正常に動作するか検証する。

- 背景: GNOME 47 へのアップデート後、英数・かなキーによる IME 切り替えが動作しないとの報告あり
- 目的: xwaykeyz が GNOME 47 の Wayland セッションで JIS キーボードのキーイベントを正常に処理できるか確認する
- 前提条件: Debian 13 がインストール済みで、Toshy セットアップ前の状態であること

## 環境情報

- OS: Debian 13 (trixie)
- DE: GNOME 47 (Wayland セッション)
- Python: 3.12
- カーネル: 6.12.x
- キーボード: Apple Magic Keyboard (JIS, A1843)
- Toshy: main ブランチ最新
- xwaykeyz: keymapper-temp/ 同梱版

## 再現方法

1. Toshy をインストール
   ```bash
   git clone https://github.com/user/toshy.git
   cd toshy
   ./setup_toshy.py install
   ```

2. systemd サービスの状態を確認
   ```bash
   systemctl --user status toshy-config toshy-session-monitor
   ```

3. キーリマッピングの動作を確認（Cmd+C → Ctrl+C、英数・かなキーで IME 切り替え等）
   ```bash
   journalctl --user -u toshy-config -f
   ```
````

## コントリビューション注意事項

- PR は `main` ブランチに対して行う
- 中〜大規模な変更はクリーンな VM スナップショットで複数ディストロ上でテスト
- xwaykeyz フォーマット: `black` (line-length 80), `isort` (profile black, width 80)
- xwaykeyz Lint: `flake8` (max-line-length 85), `pylint`, `codespell`, `bandit`
