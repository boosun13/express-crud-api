# TODO / 予約 CRUD API

Express + Zod + Prisma(SQLite) + Vitest/supertest で作る、学習用のシンプルな CRUD API。

## 技術スタック

| 用途 | ライブラリ |
|---|---|
| Web フレームワーク | Express |
| バリデーション | Zod |
| ORM / DB | Prisma + SQLite |
| テスト | Vitest + supertest |
| 実行 | TypeScript + tsx |
| Node バージョン管理 | mise（`mise.toml` で Node 24 LTS に固定） |

## 題材

**TODO → 予約** の 2 段階で進める。

- **Phase 1: TODO**：単純な CRUD で全体の流れをつかむ
- **Phase 2: 予約（Reservation）**：日時の検証、重複予約の禁止、リレーションなどの業務ルールを練習する

## ディレクトリ構成

```
src/
  app.ts              # Express アプリの組み立て（listen しない → テストで使える）
  server.ts           # app.listen() だけ
  lib/prisma.ts       # PrismaClient のシングルトン
  middlewares/
    validate.ts       # Zod スキーマで body/params/query を検証
    errorHandler.ts   # エラーを共通の JSON 形式に変換
  modules/
    todos/
      todo.schema.ts      # Zod スキーマ＋型（z.infer）
      todo.service.ts     # Prisma を使う処理
      todo.controller.ts  # req/res の処理
      todo.routes.ts
    reservations/ ...     # 同じ構成
prisma/
  schema.prisma
tests/
  setup.ts            # テスト用 DB の初期化
  todos.test.ts
```

`app.ts` と `server.ts` を分けておくと、supertest に `app` をそのまま渡せる。

## データモデル

```prisma
model Todo {
  id        Int       @id @default(autoincrement())
  title     String
  done      Boolean   @default(false)
  dueDate   DateTime?
  createdAt DateTime  @default(now())
  updatedAt DateTime  @updatedAt
}

// Phase 2
model Room {
  id           Int           @id @default(autoincrement())
  name         String        @unique
  reservations Reservation[]
}

model Reservation {
  id        Int      @id @default(autoincrement())
  roomId    Int
  room      Room     @relation(fields: [roomId], references: [id])
  guestName String
  startAt   DateTime
  endAt     DateTime
  createdAt DateTime @default(now())
}
```

## API 設計

### TODO

| Method | Path | 内容 | 成功時 |
|---|---|---|---|
| GET | `/todos?done=true` | 一覧（絞り込み） | 200 |
| GET | `/todos/:id` | 1 件取得 | 200（ない場合は 404） |
| POST | `/todos` | 作成 | 201 |
| PATCH | `/todos/:id` | 部分更新 | 200 |
| DELETE | `/todos/:id` | 削除 | 204 |

### 予約（Phase 2）

`/rooms` と `/reservations` を TODO と同じ形で追加する。

業務ルール：

- `startAt < endAt` でなければならない（Zod の `.refine`）
- 同じ部屋で時間が重なる予約は **409 Conflict**（service で判定）
- 過去の日時には予約できない

## バリデーションとエラー処理

- Zod でスキーマを 1 つ定義し、そこから**入力の検証**と **TypeScript の型**（`z.infer`）の両方を作る
- `validate({ body, params, query })` ミドルウェアで検証する。`params.id` は `z.coerce.number()` で数値に変換する
- エラーのレスポンス形式を統一する

  ```json
  { "error": { "code": "VALIDATION_ERROR", "message": "...", "details": [] } }
  ```

- `errorHandler` でまとめて変換する

  | エラー | ステータス |
  |---|---|
  | `ZodError` | 400 |
  | `NotFoundError` | 404 |
  | `ConflictError` | 409 |

## テスト方針

- 環境変数でテスト用 DB を分ける：`DATABASE_URL="file:./test.db"`
- DB の準備
  - 全テストの前（`globalSetup`）：`prisma migrate reset --force`（または `prisma db push`）
  - 各テストの前（`beforeEach`）：`deleteMany` でテーブルを空にする
- 並列実行で DB が競合しないよう、Vitest はファイル単位で直列にする（`fileParallelism: false`）
- テストの観点
  - 正常系：作成 → 取得 → 更新 → 削除の一連の流れ
  - 異常系：400（検証エラー）、404（存在しない id）、409（予約の重複）

## 進め方

細かい手順と使用バージョンは [PLAN.md](./PLAN.md) を参照。

1. [ ] プロジェクト作成（`tsx`、`typescript`、`express`、`zod`、`prisma`、`vitest`、`supertest`）
2. [ ] Prisma の初期設定と TODO モデルのマイグレーション
3. [ ] `app.ts`、エラーハンドラ、validate ミドルウェアを作る
4. [ ] TODO の CRUD とテスト
5. [ ] 予約機能（Room / Reservation）とルールのテスト
6. [ ] 発展：ページング、OpenAPI 生成（`zod-to-openapi`）、簡単なフロント画面
