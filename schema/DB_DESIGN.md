# DB設計書（PostgreSQL）

対象: ユーザー、組織、所属履歴、雇用履歴（流入・助成金はユーザー列）

## 1. 設計方針
- **時間で変化する情報は履歴テーブルで管理**  
  所属・雇用条件は `user_organization_histories` / `user_employment_histories` に集約。現在値はクエリで導出。
- **PostgreSQL ENUM を積極採用**  
  権限、雇用形態、勤務地などは ENUM でドメインを固定（誤入力防止・集計の安定化）。
- **組織の階層は自己参照**  
  `organizations.parent_organization_id` による隣接リスト方式。必要に応じて `ltree`/closure table へ拡張可能。
- **責任者は現値のみ保持**  
  `organizations.manager_user_id` で参照。責任者履歴は保持しない（運用を軽量化）。
- **整合性の責務分担**  
  DBは外部キー/ユニーク/CHECK等でドメイン整合を担保。雇用期間の重複など一部はアプリ側で検証。

## 2. 命名・型・共通ルール
- 物理名: `snake_case`
- 主キー: `BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY`
- 時刻: `TIMESTAMPTZ`（`created_at`/`updated_at` は `DEFAULT now()`）
- 文字列: 原則 `TEXT`。メールは正規化の都合で `LOWER(email)` にユニークインデックス
- 外部キーのON DELETE/UPDATEは下表参照

## 3. ENUM型（PostgreSQL）
```sql
-- ドメイン値
CREATE TYPE user_role            AS ENUM ('admin','writer','reader');
CREATE TYPE invitation_status    AS ENUM ('invited','registered');
CREATE TYPE join_category        AS ENUM ('新卒','中途');
CREATE TYPE employment_type      AS ENUM ('役員','正社員','契約社員_有期','契約社員_トライアル','アルバイト','業務委託');
CREATE TYPE work_engagement      AS ENUM ('常駐','非常駐');
CREATE TYPE work_location        AS ENUM ('東京本社','大阪支社','在宅');
CREATE TYPE working_pattern      AS ENUM ('変形労働時間制_WAL型','完全週休二日制_WLB型','シフト制');
CREATE TYPE intern_category      AS ENUM ('インターン','内定者インターン');
CREATE TYPE work_hours           AS ENUM ('9_18','10_19','フレックス','その他');
CREATE TYPE referral_source      AS ENUM ('人材紹介','リファーラル','求人広告','ハローワーク','スカウト媒体');
CREATE TYPE grant_type           AS ENUM ('人開金_人材育成支援','トライアル雇_障がい者_特開金','特開金_特定就職困難者',
                                          'キャリアアップ_正社員化','人開金_人への投資促進','人開金_事業展開等リスキリング');
```

## 4. テーブル定義
### 4.1 users
```sql
CREATE TABLE users (
  id                  BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  employee_number     TEXT NOT NULL UNIQUE,
  last_name           TEXT NOT NULL,
  first_name          TEXT NOT NULL,
  last_name_kana      TEXT NOT NULL,
  first_name_kana     TEXT NOT NULL,
  email               TEXT NOT NULL,
  role                user_role NOT NULL DEFAULT 'writer',
  invitation_status   invitation_status NOT NULL DEFAULT 'invited',
  join_confirmed      BOOLEAN NOT NULL DEFAULT FALSE,
  join_category       join_category,
  referral_source     referral_source,
  referrer_user_id    BIGINT REFERENCES users(id) ON DELETE RESTRICT,
  referral_reward_paid_at DATE,
  grants              grant_type[],
  created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
  CONSTRAINT employee_number_format CHECK (employee_number ~ '^[0-9]{4}$'),
  CONSTRAINT referral_requires_referrer CHECK (referral_source <> 'リファーラル' OR referrer_user_id IS NOT NULL)
);
-- メールの大文字小文字を吸収したユニーク
CREATE UNIQUE INDEX users_email_unique ON users (LOWER(email));
-- 紹介者参照のための補助索引
CREATE INDEX users_referrer_user_idx ON users(referrer_user_id);
```

### 4.2 organizations
```sql
CREATE TABLE organizations (
  id                        BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name                      TEXT NOT NULL,
  manager_user_id           BIGINT NOT NULL REFERENCES users(id) ON UPDATE CASCADE ON DELETE RESTRICT,
  parent_organization_id    BIGINT REFERENCES organizations(id) ON DELETE SET NULL,
  created_at                TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at                TIMESTAMPTZ NOT NULL DEFAULT now(),
  CONSTRAINT org_parent_not_self CHECK (parent_organization_id IS NULL OR parent_organization_id <> id)
);
CREATE INDEX idx_organizations_parent ON organizations(parent_organization_id);
CREATE INDEX idx_organizations_manager ON organizations(manager_user_id);
```

### 4.3 user_organization_histories
```sql
CREATE TABLE user_organization_histories (
  id               BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  user_id          BIGINT NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
  organization_id  BIGINT NOT NULL REFERENCES organizations(id) ON DELETE RESTRICT,
  assigned_at      DATE NOT NULL,
  left_at          DATE,
  created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  CONSTRAINT uoh_dates CHECK (left_at IS NULL OR left_at > assigned_at)
);
-- よく使う参照に最適化
CREATE INDEX idx_uoh_user_assigned_desc ON user_organization_histories(user_id, assigned_at DESC);
CREATE INDEX idx_uoh_org_current ON user_organization_histories(organization_id) WHERE left_at IS NULL;
CREATE INDEX idx_uoh_user_current ON user_organization_histories(user_id) WHERE left_at IS NULL;
```

### 4.4 user_employment_histories
```sql
CREATE TABLE user_employment_histories (
  id                 BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  user_id            BIGINT NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
  assigned_at        DATE NOT NULL,
  left_at            DATE,
  employment_type    employment_type NOT NULL,
  work_engagement    work_engagement NOT NULL,
  work_location      work_location NOT NULL,
  working_pattern    working_pattern NOT NULL,
  intern_category    intern_category,
  work_hours         work_hours NOT NULL,
  work_hours_detail  TEXT,
  created_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
  CONSTRAINT ueh_dates CHECK (left_at IS NULL OR left_at > assigned_at)
  -- 期間重複はアプリ側で検証（必要になれば EXCLUDE USING GIST で移譲可）
);
CREATE INDEX idx_ueh_user_assigned_desc ON user_employment_histories(user_id, assigned_at DESC);
CREATE INDEX idx_ueh_user_current ON user_employment_histories(user_id) WHERE left_at IS NULL;
```

### 4.5 （廃止）referrals / user_grants
流入（`referral_source`/`referrer_user_id`/`referral_reward_paid_at`）と助成金（`grants` = `grant_type[]`）は `users` テーブルの列として保持します。

## 5. リレーションと削除ポリシ
- users → organizations（`manager_user_id`）: ON DELETE RESTRICT  
  責任者削除時は事前に委譲必須（責任者不在防止）。
- organizations → organizations（`parent_organization_id`）: ON DELETE SET NULL  
  親削除で子は最上位化。アプリでは子がいる削除を抑止する運用。
- organizations/users → 履歴系: ON DELETE RESTRICT  
  歴史の整合性維持を優先。履歴を残す運用とする。
- users → users（`referrer_user_id` 自己参照）: ON DELETE RESTRICT  
  紹介者ユーザー削除前に参照の付け替えが必要。

## 6. 代表クエリ
- 現在所属（ユーザー）
```sql
SELECT uoh.*
FROM user_organization_histories uoh
WHERE uoh.user_id = $1 AND uoh.left_at IS NULL;
```
- 現在所属メンバー数（組織）
```sql
SELECT COUNT(*) AS member_count
FROM user_organization_histories
WHERE organization_id = $1 AND left_at IS NULL;
```
- 最新の雇用レコード（ユーザー）
```sql
SELECT ueh.*
FROM user_employment_histories ueh
WHERE ueh.user_id = $1
ORDER BY assigned_at DESC
LIMIT 1;
```
- 階層（部分木）取得（再帰CTE）
```sql
WITH RECURSIVE org_tree AS (
  SELECT id, name, parent_organization_id, 0 AS depth
  FROM organizations WHERE id = $1
  UNION ALL
  SELECT o.id, o.name, o.parent_organization_id, t.depth + 1
  FROM organizations o
  JOIN org_tree t ON o.parent_organization_id = t.id
)
SELECT * FROM org_tree ORDER BY depth, id;
```

## 7. マイグレーション順序
1) ENUM型作成（3章）  
2) テーブル作成（4章）  
3) インデックス作成（各章末尾）  
4) 既存データ移行（必要な場合）  

## 8. 今後の拡張ポイント
- 階層の検索負荷が高い場合は `ltree` または closure table の併用
- 履歴の集計が重い場合はマテリアライズド・ビューやキャッシュで最適化
- 監査が厳格な場合は `deleted_at` による論理削除・操作ログの整備


