# システムアーキテクチャ設計書（Draft）

ピッキング効率化アプリの **全体アーキテクチャ** を定義するドキュメント。  
開発・運用・将来拡張を見据えた構成を共有し、基本設計・詳細設計の共通前提とする。

---

## 1. システム全体概要

### 1.1 ゴール

- 倉庫現場（ピッカー／パッカー）の日常業務を支える **業務 Web アプリケーション** を構築する。
- 中規模倉庫（1日あたり出荷 100〜500 行程度）を対象とし、**シンプルで保守しやすい構成** を優先する。
- 初期は社内ネットワーク／VPN 内での利用を想定し、将来的なクラウド展開も可能とする。

### 1.2 アーキテクチャの基本方針

- Web ブラウザ（PC／タブレット／スマホ）からアクセスする **3 層構造の業務システム**
- サーバサイドは **Django (Python)** を採用し、HTML レンダリング + シンプルな JSON API を提供
- DB は開発環境では SQLite、本番では PostgreSQL を想定
- ロールごとに UI と権限を分離（本社 / ピッカー / パッカー / 管理者）

---

## 2. 論理アーキテクチャ

### 2.1 3 層構成（論理ビュー）

| 層                   | 主なコンポーネント                                        | 役割                                                                         |
| -------------------- | --------------------------------------------------------- | ---------------------------------------------------------------------------- |
| プレゼンテーション層 | Picker UI / Packer UI / Admin UI（HTML + Bootstrap + JS） | ユーザーとの接点。ブラウザで画面を表示し、フォーム送信・API 呼び出しを行う。 |
| アプリケーション層   | Django アプリケーション（warehouse など）                 | ビジネスロジック・バリデーション・権限チェック・API 実装。                   |
| データ層             | RDB（SQLite / PostgreSQL）、Django ORM モデル             | 顧客・商品・注文・ピッキングリスト・検品ログ等の永続化。                     |

---

### 2.2 コンポーネント構成（論理）

- **Django プロジェクト `config`**
  - グローバル設定（settings.py, urls.py, wsgi/asgi.py）
  - 認証／認可設定
  - ロギング・ミドルウェア設定

- **アプリケーション `warehouse`**
  - 倉庫現場向け UI（テンプレート + View）
  - ピッキング／梱包／検品用 API（Django REST Framework またはシンプルな JSON View）
  - モデル（Order, OrderItem, PickingList, PickingListItem, DoubleCheckSession, DoubleCheckItem, Location, AuditLog など）
  - URL ルーティング（`/warehouse/...`, `/api/...`）

- **Django Admin**
  - 本社・管理者向けマスタ管理 UI
  - Customer, Product, Location, Order 等の CRUD 管理
  - CSV インポート機能のトリガー（管理画面からのアップロードなど）

---

## 3. 物理／デプロイ構成（初期案）

### 3.1 開発環境

- ローカル PC（Windows）
  - Python + virtualenv
  - Django 開発サーバ（`python manage.py runserver`）
  - SQLite をローカルファイルで利用
- GitHub
  - リポジトリ：`picking-efficiency-app`
  - ブランチ戦略：`main`（安定）、`feature/*`（機能別）
  - ドキュメント管理：`docs/*.md`（要件・設計）

### 3.2 本番環境（想定）

- アプリケーションサーバ
  - Linux (例: Ubuntu)
  - Python + Django
  - WSGI/ASGI サーバ（例：gunicorn, uvicorn + gunicorn）
- Web サーバ／リバースプロキシ
  - nginx
  - HTTPS 終端
- データベースサーバ
  - PostgreSQL
- ネットワーク
  - 社内 LAN or VPN 経由アクセス
  - 将来的にクラウド（AWS/Azure/GCP）上への移設も可能な構成

※ Phase1 ではローカル or 単一サーバ構成での動作確認をゴールとし、  
　本番運用構成は別途インフラ設計時に詳細化する。

---

## 4. URL / 役割分離の方針

### 4.1 ロール別 URL 範囲

| ロール       | 主な URL プレフィックス | 備考                                 |
| ------------ | ----------------------- | ------------------------------------ |
| 本社／管理者 | `/admin/`               | Django Admin 利用                    |
| ピッカー     | `/warehouse/picking/`   | ピッキング関連画面                   |
| パッカー     | `/warehouse/packing/`   | 梱包／検品関連画面                   |
| 共通認証     | `/auth/`                | ログイン／ログアウト等               |
| API（共通）  | `/api/`                 | JSON API（ピック／検品スキャンなど） |

### 4.2 API と HTML View の共存

- 同じ機能に対して：
  - **HTML View**：ブラウザ表示用（テンプレートレンダリング）
  - **API**：バックグラウンド更新用（バーコードスキャンなど）
- 例：
  - ピッキング実行画面（HTML）：`GET /warehouse/picking/<id>/`
  - ピッキングスキャン API（JSON）：`POST /api/picking/scan/`

---

## 5. データフロー（主要ユースケース）

### 5.1 ピッキング実行フロー

1. **本社側**  
   - 受注 CSV を管理画面 or スクリプトからインポート  
   - Order / OrderItem / Product / Location にデータが登録される

2. **ピッキングリスト生成（バッチ or 手動）**
   - OrderItem を集約し、PickingList / PickingListItem を生成
   - Location.sort_order に基づき棚順に並び替え

3. **ピッカー UI**
   - `/warehouse/picking/` でピッキングリスト一覧を表示
   - `/warehouse/picking/<id>/` で対象リストの明細を表示

4. **バーコードスキャン**
   - スキャンイベント → `/api/picking/scan/` に POST
   - サーバ側で PickingListItem を特定し、`qty_picked` を更新
   - 更新結果を JSON で返却し、画面の数値・進捗バーを更新

5. **ピッキング完了**
   - サーバ側で全明細が `qty_picked == qty_to_pick` になったら  
     → PickingList.status = `picking_done` に自動更新

---

### 5.2 梱包前ダブルチェック〜注文別検品フロー

1. **梱包対象リスト一覧**
   - `/warehouse/packing/picking_lists/`  
   - `status = "picking_done"` のリストを表示

2. **梱包前ダブルチェック**
   - `/warehouse/packing/<id>/precheck/` 画面  
   - `/api/packing/picking-lists/{id}/precheck-summary/` でデータ取得  
   - パッカーが目視照合し、「確認完了」 →  
     `/api/packing/picking-lists/{id}/precheck-complete/` を呼び出し  
   - PickingList.status = `packing_ready` に更新

3. **注文別検品**
   - `/warehouse/packing/picking_lists/<id>/orders/` で注文明細一覧  
   - `/warehouse/packing/order/<order_id>/` で注文ごとの検品画面  
   - バーコードスキャン → `/api/packing/scan/`  
   - `qty_verified` を更新し、全一致で Order.status = `verified`

4. **リスト完了**
   - すべての Order.status = `verified` になったら  
     → PickingList.status = `done` に更新  
   - 基幹システム側に「出荷準備完了」として連携（将来）

---

## 6. 技術スタックの前提

### 6.1 サーバサイド

- Python 3.x
- Django 4.x 予定
-（必要に応じて）Django REST Framework

### 6.2 フロントエンド

- HTML5 + CSS3
- Bootstrap 5（レスポンシブデザイン）
- シンプルな Vanilla JS or jQuery 程度（バーコード入力のフォーカス制御など）

### 6.3 データベース

- 開発環境：SQLite
- 本番環境：PostgreSQL 14 以降を想定

---

## 7. 非機能要件との関係

- 性能：
  - 1 操作あたりの API 応答時間：1 秒未満を目標
  - 同時接続 10〜30 ユーザーを想定

- 信頼性：
  - すべての更新系 API はトランザクション管理（atomic）を行う
  - ピック／検品操作は AuditLog に記録し、後追いできるようにする

- セキュリティ：
  - HTTPS + 社内ネットワーク前提
  - ロールによる権限制御（ピッカー／パッカーは実績のみ更新可）

---

## 8. 将来拡張の方向性（メモ）

- PWA 化によるオフライン時の一時バッファリング
- ネイティブアプリとの連携（API はそのまま再利用）
- 外部基幹システムとの API 連携（CSV からの段階的移行）
- 倉庫レイアウトの可視化（ロケーションマップ画面）
- マルチ倉庫・マルチテナント対応

---

## 9. 関連ドキュメント

- `project_requirements.md`（要件定義）
- `csv-format.md`（CSV インポート仕様）
- `location-design.md`（棚番／ロケーション設計）
- `api-design.md`（API 設計）
- `data-model.md`（データモデル／テーブル設計：今後作成）
