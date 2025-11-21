# データモデル設計書（Draft）

本ドキュメントは、ピッキング効率化アプリの Django モデル（データモデル）を定義する。
マスタデータ、トランザクションデータ、検品・監査ログデータの3カテゴリで整理する。

---

# 1. マスタデータ
## Customer（マスタ）

| フィールド名 | 型             | 説明                             |
| ------------ | -------------- | -------------------------------- |
| code         | CharField (PK) | 顧客コード（CSV: customer_code） |
| name         | CharField      | 顧客名（CSV: customer_name）     |

## Product（マスタ）

| フィールド名  | 型                  | 説明                                 |
| ------------- | ------------------- | ------------------------------------ |
| code          | CharField (PK)      | 商品コード（CSV: product_code）      |
| name          | CharField           | 商品名（CSV: product_name）          |
| barcode       | CharField           | JANコード等（CSV: product_barcode）  |
| location_code | CharField           | 初期棚番コード（CSV: location_code） |
| image_url     | URLField (Optional) | 誤ピック防止のための商品画像URL      |

## Location（マスタ）

| フィールド名 | 型             | 説明                                     |
| ------------ | -------------- | ---------------------------------------- |
| code         | CharField (PK) | ロケーションコード（CSV: location_code） |
| display_name | CharField      | 現場で見せる棚番名（例: A-01-05）        |
| sort_order   | IntegerField   | ピッキング時のルート順序（昇順でソート） |
| is_active    | BooleanField   | 有効/無効フラグ                          |


---

# 2. トランザクションデータ
## Order（トランザクション）

| フィールド名 | 型         | 説明                          |
| ------------ | ---------- | ----------------------------- |
| order_number | CharField  | 受注番号（CSV: order_number） |
| customer     | ForeignKey | Customer への外部キー         |
| order_date   | DateField  | 受注日（CSV: order_date）     |
| status       | CharField  | new, verified, canceled など  |

## OrderItem（トランザクション）

| フィールド名 | 型                   | 説明                                     |
| ------------ | -------------------- | ---------------------------------------- |
| order        | ForeignKey           | Order への外部キー                       |
| line_no      | IntegerField         | 明細行番号（CSV: order_line_no）         |
| product      | ForeignKey           | Product への外部キー                     |
| quantity     | IntegerField         | 受注数量（CSV: quantity / qty_expected） |
| remarks      | TextField (Optional) | 備考（CSV: remarks）                     |

## PickingList（トランザクション）

| フィールド名    | 型                | 説明                                                |
| --------------- | ----------------- | --------------------------------------------------- |
| list_id         | CharField (PK)    | ピッキングリストID (システム自動生成)               |
| orders          | ManyToManyField   | このリストに含まれる Order 群                       |
| picker          | ForeignKey (User) | 担当ピッカー (auth.User)                            |
| packer_precheck | ForeignKey (User) | 梱包前ダブルチェック実行者 (auth.User)              |
| status          | CharField         | new, in_progress, picking_done, packing_ready, done |
| created_at      | DateTimeField     | 作成日時                                            |

## PickingListItem（トランザクション）

| フィールド名  | 型                       | 説明                                              |
| ------------- | ------------------------ | ------------------------------------------------- |
| picking_list  | ForeignKey               | PickingList への外部キー                          |
| order_item    | ForeignKey               | OrderItem への外部キー                            |
| location_code | CharField                | ピック対象棚番（Product.location_codeからコピー） |
| sort_order    | IntegerField             | Location.sort_order（ルート案内用）               |
| qty_to_pick   | IntegerField             | ピッキング予定数（OrderItem.quantity）            |
| qty_picked    | IntegerField (Default 0) | ピッキング実績数                                  |
| is_complete   | BooleanField             | ピック完了フラグ (qty_picked == qty_to_pick)      |


---

# 3. 検品・監査ログデータ
## DoubleCheckSession（検品/ログ）

| フィールド名 | 型                | 説明             |
| ------------ | ----------------- | ---------------- |
| order        | OneToOneField     | 検品対象の Order |
| packer       | ForeignKey (User) | 検品担当パッカー |
| start_time   | DateTimeField     | 検品開始日時     |
| end_time     | DateTimeField     | 検品完了日時     |

## DoubleCheckItem（検品/ログ）

| フィールド名 | 型                       | 説明                             |
| ------------ | ------------------------ | -------------------------------- |
| session      | ForeignKey               | DoubleCheckSession への外部キー  |
| order_item   | ForeignKey               | OrderItem への外部キー           |
| qty_expected | IntegerField             | 検品予定数（OrderItem.quantity） |
| qty_verified | IntegerField (Default 0) | 検品実績数                       |

## AuditLog（検品/ログ）

| フィールド名   | 型                | 説明                                                            |
| -------------- | ----------------- | --------------------------------------------------------------- |
| user           | ForeignKey (User) | 操作ユーザー                                                    |
| timestamp      | DateTimeField     | 操作日時                                                        |
| action_type    | CharField         | PICK_SCAN, VERIFY_SCAN, PRECHECK_DONE など                      |
| target_model   | CharField         | 操作対象モデル名（例: PickingListItem）                         |
| target_id      | CharField         | 操作対象ID                                                      |
| change_details | JSONField         | 変更内容（例: {"qty_picked_before": 0, "qty_picked_after": 1}） |


---

# 4. リレーション概要（ER 図テキスト表現）

- Customer 1 --- * Order
- Order 1 --- * OrderItem
- Product 1 --- * OrderItem （商品は複数の受注明細で使用される）
- Location 1 --- * Product （1商品につき 1 ロケーションコード想定）
- Order 1 --- 1 DoubleCheckSession （注文ごとに1つの検品セッション）
- DoubleCheckSession 1 --- * DoubleCheckItem （セッション内の明細ごとに1レコード）
- OrderItem 1 --- 0..1 DoubleCheckItem （1明細が検品されないケースも将来想定）
- 各種更新操作 --- * AuditLog （すべての操作は監査ログに記録）

---

# 5. Django models.py へのマッピング（メモ）

実装時には上記フィールド定義をもとに、`warehouse/models.py` に Django モデルとして定義する。
主キー（PK）は、特に指定がなければ Django の自動採番 `id` を用い、業務上のキー（code, order_number など）はユニーク制約を付与する。