# 要件定義書・基本設計書

## 1. 要件定義書

### 1.1 目的

複数のシステム開発プロジェクトにおいて発生する各種ドキュメント（要件定義書、設計書、試験仕様書等）を、**GitHub を正本（Single Source of Truth）**として一元管理し、

- 作成中版／正式版の明確な区別
- 権限に基づく閲覧・編集・承認統制
- 変更履歴・監査性の確保
- AI（ChatGPT / Gemini）による支援
  を実現することを目的とする。

### 1.2 背景・課題

- プロジェクト横断でドキュメント形式・品質が揃わない
- 正式版とドラフトの混在による誤利用リスク
- レビュー・承認プロセスが属人化
- GitHub は使っているが非エンジニアには扱いづらい

### 1.3 想定利用者

| 区分 | 想定ユーザ | 主な操作 |
| --- | --- | --- |
| プラットフォーム管理者 | 管理者 | 全体管理、権限付与 |
| プロジェクト管理者 | PM | プロジェクト管理、承認 |
| 編集者 | 設計担当 | 文書作成・更新 |
| レビュア | QA/有識者 | レビュー・承認 |
| 閲覧者 | 顧客/関係者 | 閲覧のみ |

### 1.4 対象ドキュメント

- 要件定義書
- 基本設計書 / 詳細設計書
- 試験仕様書 / 結果報告書
- 運用設計書
- その他 Markdown / PDF / Office 文書

### 1.5 スコープ

#### 対象

- Web UI による操作
- GitHub リポジトリ連携
- Keycloak による認証・認可
- Docker によるローカル構築

#### 対象外（初期）

- 商用SaaS連携（Jira 等）
- 大規模ワークフローエンジン

### 1.6 機能要件

#### 1.6.1 認証・認可

- Keycloak によるログイン
- ロールベース権限制御（admin/editor/reviewer/viewer）

#### 1.6.2 プロジェクト管理

- プロジェクト作成・一覧
- メンバー招待・ロール付与

#### 1.6.3 ドキュメント管理

- 文書登録（ドラフト）
- 作成中版 / 正式版の状態管理
- GitHub ブランチ／タグ連携

#### 1.6.4 レビュー・承認

- レビュー依頼
- 承認操作（正式版確定）

#### 1.6.5 AI支援

- 文書構成チェック
- 記載漏れ検出
- 要約生成

#### 1.6.6 監査・履歴

- 操作ログ（誰が・いつ・何を）
- GitHub コミット履歴参照

### 1.7 非機能要件

- ローカル環境で完結（Docker）
- GitHub API 制限内で動作
- 日本語文書を前提

---

## 2. 基本設計書

### 2.1 全体構成

```
[Browser]
   │
   ▼
[Web UI (React)]
   │ REST
   ▼
[API (FastAPI)] ── [PostgreSQL]
   │
   ├─ GitHub API
   └─ AI API (ChatGPT / Gemini)

[Keycloak]
```

### 2.2 技術スタック

| 領域 | 技術 |
| --- | --- |
| UI | React + Vite |
| API | FastAPI (Python) |
| DB | PostgreSQL |
| 認証 | Keycloak |
| SCM | GitHub |
| AI | OpenAI API / Gemini API |
| 実行環境 | Docker Compose |

### 2.3 認証・認可設計

- OIDC（Keycloak）
- トークン検証は API 側で実施
- ロール判定：
  - プラットフォームロール（admin）
  - プロジェクトロール（editor 等）

### 2.4 データ設計（概要）

#### projects

- id
- name
- created_at

#### project_members

- project_id
- user_sub
- role

#### documents

- project_id
- doc_id
- path
- status（draft / released）

#### audit_logs

- actor
- action
- detail(JSON)
- at

### 2.5 GitHub 連携設計

- 1プロジェクト = 1リポジトリ（原則）
- draft: feature/* ブランチ
- released: main + tag

### 2.6 AI連携設計

- API 経由で Markdown を送信
- チェック結果を JSON で返却
- UI で差分・指摘を表示

### 2.7 画面設計（概要）

- ログイン画面
- プロジェクト一覧
- プロジェクト詳細（文書一覧・ロール表示）
- 文書詳細（状態・レビュー状況）

### 2.8 エラーハンドリング

- 認証エラー：401
- 権限不足：403
- GitHub API エラー：502 相当

### 2.9 運用・引継ぎ

- docker compose up で即起動
- 初期化スクリプトで Keycloak/DB を自動構築
- README に起動・制約を明記

---

## 3. 今後の拡張候補

- 承認ワークフロー可視化
- PDF自動生成
- Slack通知
- AWS移行

# 詳細設計書（概要）

## 1. 概要

本書は「AI×GitHub ドキュメント管理システム 要件定義書・基本設計書」を前提とし、**実装可能レベルまで落とし込んだ詳細設計**を示す。対象読者は以下を想定する。

- 実装担当エンジニア
- レビュー・品質管理担当
- 他AIへの引き継ぎ先

---

## 2. 機能一覧（詳細）

### 2.1 認証・認可

| ID | 機能名 | 説明 |
| --- | --- | --- |
| AUTH-01 | ログイン | Keycloak(OIDC) による認証 |
| AUTH-02 | トークン検証 | APIでJWT署名・issuer検証 |
| AUTH-03 | ロール判定 | realm role / project role を分離判定 |

---

## 3. 権限マトリクス

### 3.1 プラットフォームロール

| 機能 | admin | editor | reviewer | viewer |
| --- | --- | --- | --- | --- |
| プロジェクト作成 | ○ | × | × | × |
| 全プロジェクト閲覧 | ○ | × | × | × |

### 3.2 プロジェクト内ロール

| 機能 | admin | editor | reviewer | viewer |
| --- | --- | --- | --- | --- |
| 文書登録 | ○ | ○ | × | × |
| 文書閲覧 | ○ | ○ | ○ | ○ |
| メンバー管理 | ○ | × | × | × |
| 承認 | ○ | × | ○ | × |

---

## 4. API詳細設計

### 4.1 認証系

#### GET /health

- 用途: 死活監視
- 認証: 不要
- Response:

```json
{ "ok": true }
```

#### GET /me

- 用途: ログインユーザ情報取得
- 認証: Bearer Token

---

### 4.2 プロジェクト管理

#### POST /projects

- 権限: platform admin
- Request:

```json
{ "name": "ProjectA" }
```

#### GET /projects

- 権限: 全ロール
- 挙動:
  - admin: 全件
  - 非admin: 自分が所属するもののみ

---

### 4.3 メンバー管理

#### POST /projects/{project_id}/members

- 権限: platform admin または project admin
- Request:

```json
{ "user_sub": "xxx", "role": "viewer" }
```

#### GET /projects/{project_id}/members

- 権限: project admin

---

### 4.4 ドキュメント管理

#### POST /projects/{project_id}/docs

- 権限: admin / editor
- 用途: ドラフト登録
- Request:

```json
{ "doc_id": "SPEC-001", "path": "docs/SPEC-001.md" }
```

#### GET /projects/{project_id}/docs

- 権限: 全ロール

---

## 5. 状態遷移設計（ドキュメント）

```
[draft] --(editor作成)--> [in_review]
[in_review] --(reviewer承認)--> [released]
```

| 状態 | 説明 |
| --- | --- |
| draft | 作成中 |
| in_review | レビュー中 |
| released | 正式版 |

---

## 6. DB詳細設計

### 6.1 projects

| カラム | 型 | 備考 |
| --- | --- | --- |
| id | serial | PK |
| name | text | unique |

### 6.2 project_members

| カラム | 型 | 備考 |
| --- | --- | --- |
| project_id | int | FK |
| user_sub | text | Keycloak sub |
| role | text | admin/editor/... |

### 6.3 documents

| カラム | 型 | 備考 |
| --- | --- | --- |
| project_id | int | FK |
| doc_id | text | 論理ID |
| path | text | GitHubパス |
| status | text | draft/released |

### 6.4 audit_logs

| カラム | 型 | 備考 |
| --- | --- | --- |
| actor | text | 操作者 |
| action | text | 操作 |
| detail | jsonb | 詳細 |

---

## 7. 画面遷移設計

```
ログイン
  ↓
プロジェクト一覧
  ↓
プロジェクト詳細
  ├─ 文書一覧
  └─ メンバー一覧（adminのみ）
```

---

## 8. エラーハンドリング方針

| ケース | HTTP | 備考 |
| --- | --- | --- |
| 未認証 | 401 | トークン不正 |
| 権限不足 | 403 | ロール不足 |
| GitHub失敗 | 502 | 外部依存 |

---

## 9. 実装上の注意

- Keycloak issuer / jwks は **コンテナ名基準**
- DB 起動待ち必須
- 正式版確定は GitHub tag を正とする

---

## 10. 次フェーズへの引継ぎポイント

- ステータス遷移APIは未実装（次フェーズ）
- UIは最小構成（管理画面中心）
- AIチェックは非同期化推奨
