# 📦 倉庫ピッキング効率化アプリ（Picking Efficiency App）

倉庫業務における **ピッキング → 梱包前ダブルチェック → 注文別検品 → 出荷準備** の一連プロセスを  
デジタル化し、**作業効率化・ミス削減・品質向上**を実現する Web アプリケーションです。

本アプリは現場スタッフ（ピッカー / パッカー）が主に使用するため、  
**モバイル端末での操作性・視認性・安定性** を最優先に設計されています。

## 🎨 アプリとしてのこだわり（UI/UXへの強い思い）

本アプリは 日本の倉庫現場で働く老若男女すべてのスタッフが  
「直感的に使えて」「迷わず操作できて」「ストレスなく作業できる」ことを最優先に設計しています。

現場では、  
IT 操作が得意ではないベテラン作業者  
小さなスマホ画面で作業するピッカー  
忙しい時間帯に集中して作業するパッカー  
繁忙期や新人スタッフの即戦力化  
など、多様なユーザーが混在しています。  

そのため本アプリでは、次のこだわりを込めています。

🌟 1. 視認性を最優先にしたデザイン  

大きく、はっきりした文字・ボタン  
棚番・商品名・数量の明確なレイアウト  
色覚障がいにも配慮した配色（完了=緑、注意=黄、警告=赤 等）  
スキャン後の視線移動を最小化した画面配置  
📌 高齢者や未経験者でもミスしにくく、安心して使える設計を追求。  

🌟 2. “片手で扱える”ピッカー専用 UI  

倉庫内での移動・棚作業を想定し、  
片手での操作  
バーコードスキャン入力を自動フォーカス  
誤タップを避ける大型ボタン  
親指で操作しやすい下部メニュー／戻る動線  
など、現場の動作そのものを意識した UX 最適化を採用しています。  

🌟 3. 作業の「流れ」を止めない設計

倉庫業務はテンポが命。
そのため…  
スキャン成功 → 即時フィードバック（音/色/振動）  
誤スキャン時は一瞬で気付ける  
通信が不安定でも作業が止まらない（リトライ／オフライン仕様への拡張を考慮）  
“余計な画面遷移がゼロ” を目指した操作導線  
📌 現場の「作業リズムを壊さない」ことにこだわりました。  

🌟 4. 現場の声を吸い上げて育つアプリ

棚順変更 → Location マスタ変更だけで対応  
誤ピック率の見える化で改善サイクル  
ピッカー／パッカーの行動ログに基づく UI チューニング  
倉庫ごとの文化に合わせた柔軟カスタマイズ  

📌 作業者の声を活かして「倉庫と一緒に成長するアプリ」を目指します。  

🌟 5. シンプルで美しく、日本人にしっくりくる UI

海外アプリのような派手さではなく、  
日本人の“仕事のしやすさ”に合わせた美しさを追求。  
無駄のないミニマルUI  
業務アプリとしての落ち着いた配色  
情報量を必要十分に調整  
日本語の表記ゆれを徹底排除（例：数量/数/個数…）  
倉庫内の照明下でも見やすいコントラスト  

📌 デザインは派手さより “正確性と安心感” を重視しています。

🌟 6. 新人・助っ人スタッフでも即戦力化  
UI/UX のわかりやすさにより、  
「説明なしで使える」  
「説明書が不要」  
「今日初めて来た人でもミスしにくい」  
という、人材不足の現場で特に強い価値を発揮します。  

✨ まとめ：

“現場の人のためのアプリ”であることにこだわっています。  

倉庫のスタッフ全員が  
「これなら私でもできる」  
「これなら安心して任せられる」  
「このアプリなら作業が楽になる」  

そう思ってもらえる UI/UX を目指して設計・開発しています。  

---

## 🚀 主な特徴（Features）

- **ピッキングリストに基づく棚順案内**
- バーコードスキャンで **qty_picked / qty_verified を即時更新**
- 梱包前に “リスト全体の整合性” を確認する  
  **梱包前ダブルチェック機能（本プロジェクトの重要要件）**
- 注文単位のダブルチェック（DoubleCheckSession）
- ピッキング → 検品 → 梱包 → 出荷準備の **ステータス自動遷移**
- ロケーション（棚番）マスタによる  
  **柔軟な棚番運用（display_code／sort_order 分離）**
- Django Admin による顧客・商品・注文データ管理

---

## 🏗 技術スタック

| 項目             | 詳細                                    |
| ---------------- | --------------------------------------- |
| 言語             | Python 3.12+, JavaScript                |
| フレームワーク   | Django 4.x                              |
| データベース     | PostgreSQL（本番） / SQLite（開発）     |
| UI               | HTML5 + Bootstrap 5（レスポンシブ対応） |
| ロケーション管理 | Location マスタでコードとソート順を分離 |

---

## 📂 ディレクトリ構成

```text
picking-efficiency-app/
├── backend/                         # Djangoプロジェクト
│   ├── config/                      # 設定 (settings, urls, wsgi/asgi)
│   ├── warehouse/                   # 倉庫業務アプリ
│   │   ├── migrations/
│   │   ├── management/commands/     # カスタムコマンド
│   │   ├── models.py                # Location, PickingList, etc.
│   │   ├── views/                   # picking_views, packing_views, api_views
│   │   ├── services/                # ドメインロジック層
│   │   ├── templates/warehouse/     # HTMLテンプレート
│   │   ├── static/warehouse/        # CSS/JS/画像
│   │   └── tests/                   # 自動テスト
│   ├── accounts/                    # 認証・ロール管理（将来）
│   └── manage.py
│
├── docs/                            # 設計書・仕様書
│   ├── api-design.md
│   ├── architecture.md
│   ├── csv-format.md
│   ├── data-model.md
│   ├── location-design.md
│   └── project_requirements.md
│
├── scripts/                         # 補助スクリプト
│   ├── import_orders_from_csv.py
│   └── generate_dummy_data.py
│
├── .github/workflows/               # CI/CD（将来）
│   └── ci.yml
│
├── .env.example
├── .gitignore
├── requirements.txt
└── README.md
```

---

## 🔧 開発環境セットアップ

### 1. リポジトリのクローン

\`\`\`bash
git clone https://github.com/<username>/picking-efficiency-app.git
cd picking-efficiency-app/
\`\`\`

### 2. Python 仮想環境の作成

\`\`\`bash
python3 -m venv venv
source venv/bin/activate            # Windows: .\venv\Scripts\activate
pip install -r requirements.txt
\`\`\`

### 3. DB セットアップ

\`\`\`bash
python backend/manage.py makemigrations
python backend/manage.py migrate
python backend/manage.py createsuperuser
\`\`\`

### 4. 開発サーバー起動

\`\`\`bash
python backend/manage.py runserver
\`\`\`

---

## 🔥 GitHub Flow（開発ルール）

### ■ main ブランチ
- 本番運用ブランチ
- **直接コミット禁止**

### ■ develop ブランチ
- 開発の中心ブランチ
- Pull Request でのみマージ

### ■ feature ブランチ
- 機能追加／修正は必ず feature ブランチで作業
- 命名例：
  - \`feature/picking-ui\`
  - \`feature/csv-import\`
  - \`fix/scan-bug\`

---

## 📬 Pull Request（PR）ルール

PR 作成時には以下を必ず記載：

- **目的**：このPRが実現する仕様、改善点  
- **変更内容**：主要ファイル、処理内容  
- **テスト方法**：動作確認手順  
- **影響範囲**：他機能に影響が出る可能性

---

## 📝 コミットメッセージ規約（Conventional Commits）

| 種別     | 意味                 | 例                                |
| -------- | -------------------- | --------------------------------- |
| feat     | 新機能               | feat: add picking detail page     |
| fix      | バグ修正             | fix: prevent over-picking         |
| docs     | ドキュメント更新     | docs: update README               |
| style    | 見た目のみの変更     | style: unify mobile button size   |
| refactor | 改善（動作変更なし） | refactor: extract picking service |
| chore    | CI設定・環境構築     | chore: add postgres config        |

---

## 🗃 主要データモデル（概要）

### マスタ
- Product（商品）
- Location（棚番：display_code / sort_order）

### トランザクション
- Order / OrderItem
- PickingList / PickingListItem

### 検品
- DoubleCheckSession
- DoubleCheckItem

### 監査
- AuditLog（スキャン／ステータス遷移を記録）

---

## 📦 業務フロー（要約）

\`\`\`
1. ピッキング（qty_picked 更新）
2. ピッキング完了判定
3. 梱包前ダブルチェック（リスト全体の整合性確認）
4. 注文一覧 → 注文選択
5. 注文ごとのダブルチェック（qty_verified 更新）
6. 全注文 verified → PickingList.done → 出荷準備OK
\`\`\`

---

## 📜 ライセンス

（必要に応じて記載）
