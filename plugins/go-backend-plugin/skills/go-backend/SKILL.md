---
name: go-backend
description: Go製APIバックエンド開発ガイド。クリーンアーキテクチャに基づくAPI開発、新機能実装、テスト、デプロイの標準フロー。契約定義、スキーマ設計、ユースケース実装時に自動的に使用。
---

# Go Backend Development Skill

このスキルは、Go製バックエンドAPIの開発に必要な知識とワークフローを提供します。

## アーキテクチャ

クリーンアーキテクチャに基づく4層構造：

| 層 | ディレクトリ | 責務 |
|---|---|---|
| ドメイン | `backend/internal/domain/` | エンティティ、値オブジェクト、リポジトリインターフェース |
| ユースケース | `backend/internal/usecase/` | ビジネスロジック、トランザクション管理 |
| ゲートウェイ | `backend/internal/gateway/` | リポジトリ実装、外部システム連携 |
| プレゼンテーション | `backend/internal/presentation/api/` | HTTPハンドラー、リクエスト/レスポンス処理 |

**依存の方向**: プレゼンテーション → ユースケース → ドメイン ← ゲートウェイ

### トレース

OpenTelemetryで各層にトレースを実装。OTLP HTTPで送信。

## 新機能実装フロー

ドメインを起点とし、内から外へ層を実装する標準フロー：

### 1. ドメインエンティティの定義

`backend/internal/domain/` にエンティティ・値オブジェクトを定義。

- コンストラクタ関数で生成（例: `NewUser`, `NewOrder`）
- イミュータブル、バリデーションはコンストラクタで実行
- エラーは明示的に返却

### 2. ユースケース実装（リポジトリインターフェース定義を含む）

- `backend/internal/domain/` にリポジトリインターフェースを定義
- `backend/internal/usecase/` にビジネスロジックを実装
- リポジトリインターフェースを依存として受け取る
- トランザクション管理、エラーハンドリングの統一

### 3. API定義（OpenAPI）

- `backend/openapi.yaml` を編集
- エンドポイント・リクエスト・レスポンスを定義
- コード生成ツール（oapi-codegen等）で型とサーバースタブを生成

### 4. ハンドラー実装

- `backend/internal/presentation/api/handler/` にハンドラーを実装
- ユースケースを依存として受け取る
- 認証情報の取得（認証ミドルウェアから）
- レスポンス整形（ドメインモデル → 生成されたレスポンス型）
- サーバーへのハンドラー登録、エントリーポイントでDI

### 5. リポジトリ実装（ゲートウェイとDBマイグレーション）

- スキーマファイルにテーブル定義
- クエリファイルにSQL定義
- sqlc等でコード生成
- `backend/internal/gateway/repository/` にリポジトリ実装
- マイグレーション適用
- エントリーポイントでDI設定を更新

### 6. ビルド確認

`go vet ./...` でビルドエラーと静的解析を実行。

## 避けるべきこと

- **過度な設計**: 要求された機能以外の追加や改善を避ける
- **後方互換性ハック**: 未使用の変数名変更、型の再エクスポート、削除コードのコメントなど。不要なものは完全に削除する
- **早すぎる抽象化**: 1回限りの操作のためのヘルパーやユーティリティを作らない
- **不要なエラーハンドリング**: 発生しないシナリオの検証を追加しない。システム境界（ユーザー入力、外部API）でのみ検証する

## 開発コマンド

すべてのタスクはリポジトリルートから実行します。プロジェクトのタスクランナー（Taskfile等）に合わせて調整してください。

```bash
# データベース
# マイグレーション適用（Atlas, golang-migrate 等）
# サンプルデータ投入

# コード生成
# sqlc でクエリコード生成
# OpenAPI から型とサーバースタブ生成
# モック生成（mockgen 等）

# テスト（ユーザーから明示的な指示があった場合のみ）
# go test -race ./...
```

## ローカルデバッグ

### デバッグ手法

**1. ログベースデバッグ**

コードに `log.Printf` を追加 → ホットリロード（air等）が自動リビルド → ターミナルでログ確認

```go
log.Printf("Debug: userID=%s, amount=%d", userID, amount)
```

**2. APIテスト**

```bash
# ヘルスチェック
curl http://localhost:<port>/health

# レスポンスとステータスコード確認
curl -s -w "\nHTTP Status: %{http_code}" http://localhost:<port>/v1/<resource>
```

**3. 静的解析**

```bash
go vet ./...
```

**4. トレース確認**

Jaeger等のトレースUIでリクエストのトレースを確認。

## 重要な実務ルール

### コード生成の管理

- **API定義後**: `openapi.yaml` 変更 → コード生成コマンド実行
- **SQL定義後**: スキーマ/クエリ変更 → sqlc 生成コマンド実行
- **リポジトリインターフェース追加後**: モック生成コマンド実行
- **生成物は元データと同じコミットに含める**

### コーディング規約

- **Go**: `gofmt` / `goimports` を必ず適用（タブインデント）
- **生成ファイル** (`*.gen.go`): 手動編集は禁止。生成元を更新しコマンドを再実行する
- **エラーハンドリング**: 明示的なエラー返却

### 実装パターン

#### ドメイン層

**エンティティ**
- ファクトリ関数 (`NewXxx`, `CreateXxx`) には `context.Context` を第一引数に取る
- バリデーションエラーは `ValidationError` を使い、フィールド名 + メッセージで構造化

**リポジトリ**
- 集約単位で分割（テーブル単位ではない）
- インターフェースの引数には名前を付ける (`ctx context.Context, id ulid.ULID`)
- 関連エンティティは直接参照する（中間エンティティは作らない）

**トレーシング**
- span名は `domain.エンティティ名.操作` 形式（例: `domain.Order.Create`）
- deferパターンでエラー記録を統一

#### ユースケース層

**構造**
- `XxxUseCaseImpl` 構造体 + `NewXxxUseCase` ファクトリ関数
- 依存はリポジトリインターフェースで注入

**トレーシング**
- span名は `usecase.エンティティ名.操作` 形式（例: `usecase.Order.Create`）
- deferパターンでエラー記録を統一:
  ```go
  func (u Impl) Method(ctx context.Context, input Input) (output *Output, err error) {
      ctx, span := tracer.Start(ctx, "usecase.Xxx.Method")
      defer func() {
          if err != nil {
              span.SetStatus(codes.Error, err.Error())
              span.RecordError(err)
          }
          span.End()
      }()
      // ...
  }
  ```

#### GoDoc

- 構造体・メソッドにコメントを付ける
- ファクトリ関数は「〜のファクトリ関数」と記載
- パッケージには `doc.go` を作成し説明を記載

## 参考資料

### Context7 で最新ドキュメントを参照

ライブラリのAPIを確認する際は `use context7` を使用すること。LLMの学習データより新しい情報を取得できる。

対象ライブラリ:
- **sqlc** - クエリ構文、設定オプション
- **oapi-codegen** - OpenAPI生成設定
- **OpenTelemetry** - Go SDK トレースAPI
