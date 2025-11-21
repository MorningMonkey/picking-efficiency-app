# API 設計書（Draft）

本ドキュメントでは、ピッキング効率化アプリにおける **倉庫現場向け API** の仕様を定義する。  
主な対象は以下の業務機能である：

- ピッキングリストの取得・進捗更新
- 梱包前ダブルチェック（リスト全体照合）
- 注文別ダブルチェック（検品）
- バーコードスキャンによる実績更新

---

## 1. 共通仕様

### 1.1 ベース URL

- 開発環境: `http://localhost:8000/`
- API ベースパス: `/api/`

例：  
`GET http://localhost:8000/api/picking-lists/`

### 1.2 認証・認可

- 認証方式（初期案）：
  - Phase1: Django セッション認証（Web ログイン後の利用を前提）
  - Phase2 以降: Token 認証 or JWT を検討
- ロール：
  - Picker：ピッキング系 API のみ更新可能
  - Packer：梱包・検品系 API のみ更新可能
  - Admin：参照＋管理 API（将来）

### 1.3 リクエスト／レスポンス形式

- Content-Type: `application/json`
- 文字コード: UTF-8
- クライアントは通常 **JSON Body + HTTP Header** を利用

### 1.4 ステータスコードルール

| ステータス                | 意味                                         |
| ------------------------- | -------------------------------------------- |
| 200 OK                    | 成功（通常レスポンス）                       |
| 201 Created               | 新規作成成功                                 |
| 400 Bad Request           | パラメータ不正、バリデーションエラー         |
| 401 Unauthorized          | 未認証                                       |
| 403 Forbidden             | 権限不足（ロール不一致など）                 |
| 404 Not Found             | リソースが存在しない                         |
| 409 Conflict              | ビジネス整合性エラー（例：予定数超過ピック） |
| 500 Internal Server Error | サーバ側エラー                               |

### 1.5 エラーレスポンスの共通形式（案）

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "quantity must be greater than 0",
    "details": {
      "field": "quantity",
      "value": -1
    }
  }
}
```

---

## 2. ピッキングリスト関連 API

### 2.1 ピッキングリスト一覧取得

- **HTTP Method**: `GET`
- **URL**: `/api/picking-lists/`
- **概要**: ログインユーザーが担当可能なピッキングリストを取得する。

#### クエリパラメータ（例）

| パラメータ     | 必須 | 型     | 説明                                                               |
| -------------- | ---- | ------ | ------------------------------------------------------------------ |
| status         | 任意 | string | `new`, `in_progress`, `picking_done`, `packing_ready`, `done` など |
| assigned_to_me | 任意 | bool   | `true` の場合、ログインユーザー担当分のみ取得                      |

#### レスポンス例

```json
[
  {
    "id": 123,
    "name": "PICKLIST-20241121-01",
    "status": "new",
    "total_items": 35,
    "total_quantity": 120,
    "picked_quantity": 0,
    "assigned_to": null,
    "created_at": "2024-11-21T09:00:00+09:00"
  },
  {
    "id": 124,
    "name": "PICKLIST-20241121-02",
    "status": "in_progress",
    "total_items": 20,
    "total_quantity": 80,
    "picked_quantity": 30,
    "assigned_to": "picker001"
  }
]
```

---

### 2.2 ピッキングリスト明細取得

- **HTTP Method**: `GET`
- **URL**: `/api/picking-lists/{id}/items/`
- **概要**: 指定したピッキングリストの明細を棚番順で取得する。

#### パスパラメータ

| 名前 | 説明              |
| ---- | ----------------- |
| id   | PickingList の ID |

#### レスポンス例

```json
{
  "id": 123,
  "name": "PICKLIST-20241121-01",
  "status": "in_progress",
  "items": [
    {
      "id": 1,
      "order_number": "20241101001",
      "order_line_no": 1,
      "product_code": "P-1001",
      "product_name": "サンプル商品A",
      "product_barcode": "4901234567890",
      "location_code": "A-01-05",
      "qty_to_pick": 5,
      "qty_picked": 3
    },
    {
      "id": 2,
      "order_number": "20241101001",
      "order_line_no": 2,
      "product_code": "P-1002",
      "product_name": "サンプル商品B",
      "product_barcode": "4901234567891",
      "location_code": "A-02-03",
      "qty_to_pick": 2,
      "qty_picked": 2
    }
  ]
}
```

---

## 3. ピッキング（バーコードスキャン）API

### 3.1 ピッキングスキャン（qty_picked 更新）

- **HTTP Method**: `POST`
- **URL**: `/api/picking/scan/`
- **概要**: ピッカーが商品バーコードをスキャンした際に呼び出される API。  
  バーコード ＋ ピッキングリストID から該当行を特定し、`qty_picked` を +1 する。

#### リクエストボディ（案）

```json
{
  "picking_list_id": 123,
  "barcode": "4901234567890",
  "quantity": 1
}
```

- `quantity` は省略時 1 をデフォルトとする想定。

#### 正常時レスポンス例

```json
{
  "result": "ok",
  "picking_list_id": 123,
  "item": {
    "id": 1,
    "product_barcode": "4901234567890",
    "qty_to_pick": 5,
    "qty_picked": 4
  },
  "list_progress": {
    "total_items": 35,
    "completed_items": 10,
    "total_quantity": 120,
    "picked_quantity": 80,
    "status": "in_progress"
  }
}
```

#### バリデーション・ビジネスルール（例）

- 対象 PickingList の status が `in_progress` 以外ならエラー（409）
- 対象明細の `qty_picked + quantity > qty_to_pick` の場合はエラー（409）
- 対象バーコードが該当リスト内に存在しない場合はエラー（404 / 400）

#### エラーレスポンス例（数量超過）

```json
{
  "error": {
    "code": "PICKING_OVER_QUANTITY",
    "message": "qty_picked would exceed qty_to_pick",
    "details": {
      "qty_to_pick": 5,
      "qty_picked": 5,
      "additional": 1
    }
  }
}
```

---

### 3.2 ピッキング完了判定（オプション）

- **HTTP Method**: `POST`
- **URL**: `/api/picking-lists/{id}/complete/`
- **概要**: 明示的に「ピッキング完了」ボタンを押したときに呼び出す API。  
  実際には、サーバ側で全明細チェックを行い、整合すれば `status = "picking_done"` に更新する。

#### レスポンス例

```json
{
  "result": "ok",
  "picking_list_id": 123,
  "status": "picking_done"
}
```

---

## 4. 梱包前ダブルチェック（リスト全体照合）API

### 4.1 梱包対象リスト一覧取得

- **HTTP Method**: `GET`
- **URL**: `/api/packing/picking-lists/`
- **概要**: `status = "picking_done"` のピッキングリストを取得し、梱包前ダブルチェック候補として表示する。

#### レスポンス例（概要）

```json
[
  {
    "id": 123,
    "name": "PICKLIST-20241121-01",
    "status": "picking_done",
    "orders_count": 5,
    "total_quantity": 120
  }
]
```

---

### 4.2 梱包前ダブルチェック画面用データ取得

- **HTTP Method**: `GET`
- **URL**: `/api/packing/picking-lists/{id}/precheck-summary/`
- **概要**: リスト全明細とピック実績を返し、パッカーが目視照合できるデータを提供する。

#### レスポンス例

```json
{
  "id": 123,
  "name": "PICKLIST-20241121-01",
  "status": "picking_done",
  "items": [
    {
      "product_code": "P-1001",
      "product_name": "サンプル商品A",
      "product_barcode": "4901234567890",
      "location_code": "A-01-05",
      "qty_to_pick": 5,
      "qty_picked": 5
    },
    {
      "product_code": "P-1002",
      "product_name": "サンプル商品B",
      "product_barcode": "4901234567891",
      "location_code": "A-02-03",
      "qty_to_pick": 3,
      "qty_picked": 2
    }
  ],
  "summary": {
    "all_matched": false,
    "mismatch_count": 1
  }
}
```

---

### 4.3 梱包前ダブルチェック完了 API

- **HTTP Method**: `POST`
- **URL**: `/api/packing/picking-lists/{id}/precheck-complete/`
- **概要**: パッカーが「リスト全体の照合完了」にチェックした際に呼び出す。  
  ビジネスルールに反して明らかに NG 行がある場合は警告 or ブロック。

#### リクエストボディ（案）

```json
{
  "checked_by": "packer001",
  "force": false
}
```

#### 正常時レスポンス例

```json
{
  "result": "ok",
  "picking_list_id": 123,
  "status": "packing_ready"
}
```

---

## 5. 注文別ダブルチェック（検品）API

### 5.1 梱包用：ピッキングリスト内の注文一覧取得

- **HTTP Method**: `GET`
- **URL**: `/api/packing/picking-lists/{id}/orders/`
- **概要**: 指定ピッキングリストに含まれる Order 一覧と検品ステータスを取得する。

#### レスポンス例

```json
[
  {
    "order_id": 1001,
    "order_number": "20241101001",
    "customer_name": "山田 花子",
    "status": "new"
  },
  {
    "order_id": 1002,
    "order_number": "20241101002",
    "customer_name": "佐藤 香織",
    "status": "verified"
  }
]
```

---

### 5.2 注文別検品画面用データ取得

- **HTTP Method**: `GET`
- **URL**: `/api/packing/orders/{order_id}/`
- **概要**: 注文ごとの DoubleCheckSession / DoubleCheckItem を取得する。必要に応じてセッションを自動生成。

#### レスポンス例

```json
{
  "order_id": 1001,
  "order_number": "20241101001",
  "customer_name": "山田 花子",
  "status": "new",
  "items": [
    {
      "order_item_id": 1,
      "product_code": "P-1001",
      "product_name": "サンプル商品A",
      "product_barcode": "4901234567890",
      "qty_expected": 3,
      "qty_verified": 1
    }
  ]
}
```

---

### 5.3 検品スキャン（qty_verified 更新）

- **HTTP Method**: `POST`
- **URL**: `/api/packing/scan/`
- **概要**: パッカーが注文別検品画面でバーコードをスキャンした際に呼び出される。

#### リクエストボディ（案）

```json
{
  "order_id": 1001,
  "barcode": "4901234567890",
  "quantity": 1
}
```

#### 正常時レスポンス例

```json
{
  "result": "ok",
  "order_id": 1001,
  "item": {
    "order_item_id": 1,
    "product_barcode": "4901234567890",
    "qty_expected": 3,
    "qty_verified": 2
  },
  "order_progress": {
    "all_matched": false,
    "verified_items": 1,
    "total_items": 2
  }
}
```

#### 完全一致時の処理

- 全明細が `qty_verified == qty_expected` になった場合：
  - `Order.status = "verified"` に更新。
  - 必要であれば `DoubleCheckSession.status = "done"` に更新。

---

## 6. 監査ログ（AuditLog）連携

**対象 API すべて** で、以下の操作を AuditLog に記録する：

- ピッキングスキャン (`/api/picking/scan/`)
- 梱包前ダブルチェック完了 (`/api/packing/picking-lists/{id}/precheck-complete/`)
- 検品スキャン (`/api/packing/scan/`)
- ステータス遷移（picking_done, packing_ready, done など）

記録項目（例）：

- user_id
- operation（例：`PICK_SCAN`, `PACK_PRECHECK`, `ORDER_VERIFY`）
- target_type（`PickingList`, `Order`, `OrderItem` など）
- target_id
- payload（JSON で保存）
- created_at

---

## 7. 今後の検討項目（メモ）

- モバイル端末からのアクセスを想定した CSRF 対策・Token 認証
- レートリミット（短時間の連続スキャン対策）
- 一括スキャン（数量をまとめて加算するモード）の追加
- トランザクション境界の設計（複数 API 呼び出しの整合性）
