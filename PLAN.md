# 実装プラン

[README.md](./README.md) の構想を、上から順番に実行できる手順に分けたもの。
各ステップの最後に **✅ 確認** があるので、そこで動作を確かめてから次へ進む。

---

## 使用バージョン（2026-09-17 時点）

| 種類 | パッケージ | バージョン | 備考 |
|---|---|---|---|
| ランタイム | Node.js | **24.x LTS**（24.21.0） | 現在のLTS。26 はまだLTSになっていない |
| 言語 | typescript | 7.0.2 | |
| 実行 | tsx | 4.23.13 | TS をビルドせずに実行する |
| Web | express | 5.2.1 | |
| 型 | @types/express | 5.0.6 | |
| 型 | @types/node | 24.13.5 | Node 24 に合わせる |
| 検証 | zod | 4.6.5 | |
| ORM | prisma / @prisma/client | 7.10.0 | npm の `latest` は 8.0.0-rc（リリース候補版）なので、安定版の 7.10.0 を使う |
| SQLite | @prisma/adapter-better-sqlite3 | 7.10.0 | Prisma 7 では DB ドライバのアダプタが必須 |
| テスト | vitest | 5.0.1 | |
| テスト | supertest / @types/supertest | 7.2.2 / 7.2.1 | |

> Prisma 7 と Vitest 5 は Node 22.12 以上が必要。この PC は元々 v20.8.1 だったので、Step 0-1 で 24 LTS に上げた。

---

## Phase 0：環境準備 ✅ 完了（2026-09-17）

### Step 0-1. Node.js 24 LTS を入れる ✅
バージョン管理ツール **mise** を使う。

**なぜ mise？**
- プロジェクトのフォルダに `mise.toml` を置くと、**そのフォルダに入ったときだけ**指定のバージョンに自動で切り替わる
- PC 全体の Node（nvm-windows で管理している 20.8.1）には影響しない
- 他の人も `mise install` だけで同じバージョンをそろえられる

**手順**
1. mise をインストールする（この PC ではインストール済み）
   ```powershell
   winget install jdx.mise
   ```
2. PowerShell のプロファイル（`$PROFILE`）に有効化の設定を書く（この PC では設定済み）
   ```powershell
   (& mise activate pwsh) | Out-String | Invoke-Expression
   ```
3. プロジェクトのフォルダで Node を指定してインストールする
   ```powershell
   cd C:\study\typescript\express
   mise use node@24.21.0
   ```
   → `mise.toml` が作られる
   ```toml
   [tools]
   node = "24.21.0"
   ```
4. ✅ 確認：**ターミナルを開き直して**からプロジェクトのフォルダで
   - `node -v` → `v24.21.0`
   - `npm -v` → `11.19.0`
   - `(Get-Command node).Source` → `...\mise\installs\node\24.21.0\node.exe`

> プロジェクトの外では、今まで通り nvm-windows の Node 20.8.1 が使われる。

### Step 0-2. Git を初期化する ✅
1. `git init`（ブランチ名は `main`）
2. `.gitignore` を作る
   ```
   node_modules/
   dist/
   coverage/
   .env
   *.db
   *.db-journal
   src/generated/
   ```
3. ✅ 確認：`git status` で `node_modules` などが表示されない

### Step 0-3. VS Code を整える ✅
1. 拡張機能を入れる（`.vscode/extensions.json` に「推奨」として登録済み。フォルダを開くと入れるよう案内が出る）

   | 拡張機能 | ID | 何に使う？ |
   |---|---|---|
   | mise | `hverlin.mise-vscode` | mise の Node を VS Code でも使う |
   | Prisma | `prisma.prisma` | `schema.prisma` の色付け・補完・整形 |
   | Vitest | `vitest.explorer` | テストを画面から実行・デバッグ |
   | Prettier | `esbenp.prettier-vscode` | 保存時にコードを整形 |
   | Error Lens | `usernamehw.errorlens` | エラーをその行に直接表示 |
   | REST Client | `humao.rest-client` | `.http` ファイルから API を手動で叩く |
   | SQLite Viewer | `qwtel.sqlite-viewer` | `dev.db` の中身を見る |

2. `.vscode/settings.json` でプロジェクト用の設定をする
   - 保存時に整形（`.prisma` は Prisma 拡張、それ以外は Prettier）
   - 改行コードを LF にそろえる
   - 自動 import で `.js` 拡張子を付ける（ESM のため）
   - `node_modules`・`dist`・`src/generated` を検索から外す
3. `.gitattributes` で Git 上の改行コードも LF にそろえる
4. ✅ 確認：`code --list-extensions` に上の 7 つが表示されること

---

## Phase 1：プロジェクトの土台

### Step 1-1. package.json を作る
1. `npm init -y`
2. `package.json` に次を追加する
   - `"type": "module"`（ESM で書く）
   - `"engines": { "node": ">=24" }`
3. ✅ 確認：`package.json` が作られている

### Step 1-2. パッケージをインストールする（バージョン固定）
```powershell
npm i express@5.2.1 zod@4.6.5 @prisma/client@7.10.0 @prisma/adapter-better-sqlite3@7.10.0
npm i -D typescript@7.0.2 tsx@4.23.13 prisma@7.10.0 vitest@5.0.1 supertest@7.2.2 @types/express@5.0.6 @types/node@24.13.5 @types/supertest@7.2.1
```
✅ 確認：`npm ls --depth=0` でエラーが出ない

### Step 1-3. TypeScript の設定
1. `tsconfig.json` を作り、主に次を設定する
   - `"target": "ES2024"`
   - `"module": "NodeNext"`、`"moduleResolution": "NodeNext"`
   - `"strict": true`
   - `"outDir": "dist"`、`"rootDir": "."`
   - `"include": ["src", "tests", "prisma.config.ts", "vitest.config.ts"]`
2. ✅ 確認：`npx tsc --noEmit` がエラーなしで終わる（ファイルがまだない警告は無視してよい）

### Step 1-4. npm scripts を追加する
| script | コマンド | 用途 |
|---|---|---|
| `dev` | `tsx watch src/server.ts` | 開発サーバー（保存で再起動） |
| `build` | `tsc` | ビルド |
| `start` | `node dist/src/server.js` | ビルド後の起動 |
| `typecheck` | `tsc --noEmit` | 型チェックだけ |
| `test` | `vitest run` | テストを 1 回実行 |
| `test:watch` | `vitest` | テストを監視実行 |
| `db:migrate` | `prisma migrate dev` | マイグレーション作成と適用 |
| `db:studio` | `prisma studio` | DB を GUI で見る |

✅ 確認：`npm run` で一覧が表示される

---

## Phase 2：Express の最小構成

### Step 2-1. `src/app.ts` を作る
1. `express()` でアプリを作る
2. `app.use(express.json())` を入れる
3. 動作確認用に `GET /health` → `{ "status": "ok" }` を追加
4. `app` を `export` する（**ここでは listen しない**）

### Step 2-2. `src/server.ts` を作る
1. `app` を import して `app.listen(3000)` するだけ
2. ポートは `process.env.PORT ?? 3000`

### Step 2-3. 動かす
1. `npm run dev`
2. ✅ 確認：別ターミナルで
   ```powershell
   curl.exe http://localhost:3000/health
   ```
   → `{"status":"ok"}`

---

## Phase 3：テスト環境（先に作っておく）

### Step 3-1. `vitest.config.ts` を作る
- `test.environment: "node"`
- `test.fileParallelism: false`（SQLite の競合を防ぐ）
- `test.globalSetup: "./tests/globalSetup.ts"`（Phase 4 で使う）
- `test.env: { DATABASE_URL: "file:./test.db" }`

### Step 3-2. 最初のテストを書く
1. `tests/health.test.ts` を作る
2. `supertest(app).get("/health")` が 200 と `{ status: "ok" }` を返すことを確認
3. ✅ 確認：`npm test` が緑（1 passed）

---

## Phase 4：Prisma と SQLite

### Step 4-1. Prisma を初期化する
1. `npx prisma init --datasource-provider sqlite`
2. できるもの：`prisma/schema.prisma`、`prisma.config.ts`、`.env`
3. `.env` を `DATABASE_URL="file:./dev.db"` にする

### Step 4-2. schema.prisma を書く
1. generator を Prisma 7 の形にする
   ```prisma
   generator client {
     provider = "prisma-client"
     output   = "../src/generated/prisma"
   }
   ```
2. `Todo` モデルを追加する（README の「データモデル」参照）

### Step 4-3. マイグレーションする
1. `npm run db:migrate -- --name init_todo`
2. `npx prisma generate`（Prisma 7 は migrate 時に自動生成されないので明示的に実行する）
3. ✅ 確認
   - `prisma/migrations/` にフォルダができている
   - `src/generated/prisma/` にクライアントができている
   - `npm run db:studio` で Todo テーブルが見える

### Step 4-4. `src/lib/prisma.ts` を作る
1. `PrismaBetterSqlite3` アダプタを `DATABASE_URL` で作る
2. `new PrismaClient({ adapter })` を 1 つだけ作って export する（シングルトン）

### Step 4-5. テスト用 DB の準備
1. `tests/globalSetup.ts`：全テストの前に `prisma migrate reset --force` を `DATABASE_URL=file:./test.db` で実行する
2. `tests/helpers/resetDb.ts`：`prisma.todo.deleteMany()` を呼ぶ関数を作る
3. ✅ 確認：`npm test` を実行すると `test.db` が作られ、テストは緑のまま

---

## Phase 5：共通部品（エラー処理と検証）

### Step 5-1. 独自エラークラス
`src/errors.ts` に作る
- `AppError`（`status`、`code` を持つ基底クラス）
- `NotFoundError`（404 / `NOT_FOUND`）
- `ConflictError`（409 / `CONFLICT`）

### Step 5-2. `middlewares/errorHandler.ts`
1. `ZodError` → 400 / `VALIDATION_ERROR`（`details` に `error.issues`）
2. `AppError` → その `status` と `code`
3. それ以外 → 500 / `INTERNAL_ERROR`（ログを出す）
4. `app.ts` の**一番最後**に `app.use(errorHandler)` を置く

> Express 5 では async ハンドラ内で throw したエラーも自動で errorHandler に届く。`try/catch` や `express-async-handler` は不要。

### Step 5-3. `middlewares/validate.ts`
1. `validate({ body?, params?, query? })` の形で Zod スキーマを受け取る
2. `schema.parse()` した結果を `req.body` などに入れ直す
   - Express 5 では `req.query` が読み取り専用なので、`res.locals.query` など別の場所に入れる
3. ✅ 確認：`/health` に適当なスキーマを一時的に付けて、間違った入力で 400 が返ることをテストで確認（確認後は消す）

---

## Phase 6：TODO API（1 機能ずつ「テスト → 実装」）

各 Step は **① テストを書く → ② 失敗を確認 → ③ 実装 → ④ 緑になるのを確認** の順で進める。

### Step 6-1. スキーマを定義する（`todo.schema.ts`）
- `createTodoSchema`：`title`（1〜100文字、必須）、`dueDate`（ISO 日時、任意）
- `updateTodoSchema`：`createTodoSchema.partial()` ＋ `done`（boolean、任意）
- `idParamSchema`：`{ id: z.coerce.number().int().positive() }`
- `listQuerySchema`：`done`（`"true" | "false"` を boolean に変換、任意）
- 型は `z.infer` で作る

### Step 6-2. POST `/todos`（作成）
- テスト
  - 正しい入力 → 201、レスポンスに `id` がある
  - `title` が空 → 400 / `VALIDATION_ERROR`
- 実装：schema → service の `create` → controller → routes → `app.ts` に `app.use("/todos", todoRouter)`

### Step 6-3. GET `/todos/:id`（1 件取得）
- テスト
  - 作成した id → 200
  - 存在しない id → 404
  - `abc` のような id → 400
- 実装：`findUnique` で見つからなければ `NotFoundError` を throw

### Step 6-4. GET `/todos`（一覧）
- テスト
  - 2 件作成 → 2 件返る
  - `?done=true` → 完了のものだけ返る
- 実装：`findMany({ where, orderBy: { createdAt: "desc" } })`

### Step 6-5. PATCH `/todos/:id`（更新）
- テスト
  - `{ done: true }` → 200、`done` が true
  - 存在しない id → 404
  - 空の body `{}` → 400（何も更新しないのはエラーにする）

### Step 6-6. DELETE `/todos/:id`（削除）
- テスト
  - 削除 → 204、その後 GET すると 404
  - 存在しない id → 404

### Step 6-7. 通しのテスト
- 作成 → 一覧 → 更新 → 取得 → 削除 を 1 つのテストで流す
- ✅ 確認
  - `npm test` が全部緑
  - `npm run typecheck` がエラーなし
  - `npm run dev` で起動し、curl で手動でも一通り動く

### Step 6-8. コミット
`git commit -m "feat: TODO CRUD API"`

---

## Phase 7：予約 API

### Step 7-1. モデル追加とマイグレーション
1. `Room` と `Reservation` を `schema.prisma` に追加
2. `npm run db:migrate -- --name add_reservation` → `npx prisma generate`
3. `resetDb` に `reservation.deleteMany()` → `room.deleteMany()` を追加（**子テーブルから先に**消す）
4. ✅ 確認：`npm test` で既存の TODO テストが緑のまま

### Step 7-2. Room API
- `POST /rooms`、`GET /rooms`、`GET /rooms/:id` を TODO と同じ手順で作る
- `name` の重複 → 409（Prisma のエラーコード `P2002` を `ConflictError` に変換）

### Step 7-3. 予約のスキーマ
- `roomId`、`guestName`、`startAt`、`endAt`
- `.refine` で `startAt < endAt` をチェック
- `.refine` で `startAt` が現在より未来かチェック

### Step 7-4. POST `/reservations`（ここが一番の山場）
- テスト
  - 正常 → 201
  - `endAt <= startAt` → 400
  - 過去の日時 → 400
  - 存在しない `roomId` → 404
  - **時間が重なる予約** → 409
  - **ちょうど隣接する予約**（前の `endAt` = 次の `startAt`）→ 201（重なりではない）
- 実装：重なりの条件は「`既存.startAt < 新.endAt` かつ `既存.endAt > 新.startAt`」
  - 判定と作成は `prisma.$transaction` の中で行う

### Step 7-5. 残りの CRUD
- `GET /reservations?roomId=1`、`GET /reservations/:id`、`DELETE /reservations/:id`
- `PATCH /reservations/:id`：日時を変える場合は、**自分自身を除いて**重なりを判定する（テストも書く）

### Step 7-6. 仕上げ
- ✅ 確認：`npm test`、`npm run typecheck` がすべて通る
- `git commit -m "feat: reservation API"`

---

## Phase 8：発展（任意）

好きなものから選ぶ。

- [ ] ページング（`?page=1&limit=20`）とレスポンスへの `total` 追加
- [ ] `zod-to-openapi` で OpenAPI を生成し、Swagger UI で表示
- [ ] `vitest --coverage` でカバレッジを確認
- [ ] ESLint / Prettier の導入
- [ ] GitHub Actions で push のたびに `npm test` を実行
- [ ] 簡単なフロント画面（素の HTML + fetch）

---

## 困ったときのチェックリスト

| 症状 | よくある原因 |
|---|---|
| `Cannot find module '../generated/prisma'` | `npx prisma generate` を実行していない |
| `PrismaClient needs an adapter` | Prisma 7 はアダプタ必須。`src/lib/prisma.ts` を確認 |
| テストが時々失敗する | テストが並列で動いている → `fileParallelism: false` を確認 |
| テストで開発用のデータが消えた | `DATABASE_URL` がテスト用（`test.db`）になっていない |
| `req.query` に代入するとエラー | Express 5 では読み取り専用。`res.locals` を使う |
| import でエラー（ESM） | 相対 import に拡張子 `.js` が付いていない（`NodeNext` では必須） |
