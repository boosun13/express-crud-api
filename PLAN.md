# 実装プラン

[README.md](./README.md) の構想を、**人間が手作業で進める**想定で手順に分けたもの。

進め方の方針：

- **小さく動かす → 確かめる → 必要になったものを足す**
- パッケージや設定は最初に全部そろえず、**使う段階で入れる**
- パッケージは基本的に**バージョンを指定せずに入れる**（入ったバージョンは `package-lock.json` に記録される）
- 各 Phase の最後にコミットする

---

## バージョンについて（2026-09-17 時点）

基本は `npm i <パッケージ名>` で最新を入れればよい。ただし次の 2 つだけは**指定が必要**。

| パッケージ | 入れ方 | 理由 |
|---|---|---|
| `@types/node` | `@types/node@24` | 指定しないと Node 24 と合わない版が入ることがある |
| `prisma` / `@prisma/client` / `@prisma/adapter-better-sqlite3` | `@7` を付ける | npm の `latest` が 8.0.0-rc（リリース前の候補版）になっているため |

参考：確認時点の最新安定版

| パッケージ | バージョン |
|---|---|
| Node.js | 24.21.0（LTS） |
| typescript | 7.0.2 |
| tsx | 4.23.13 |
| express / @types/express | 5.2.1 / 5.0.6 |
| zod | 4.6.5 |
| prisma 関連 | 7.10.0 |
| vitest | 5.0.1 |
| supertest / @types/supertest | 7.2.2 / 7.2.1 |

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
   generated/
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
   - `node_modules`・`dist`・`generated` を検索から外す
3. `.gitattributes` で Git 上の改行コードも LF にそろえる
4. ✅ 確認：`code --list-extensions` に上の 7 つが表示されること

---

## Phase 1：Express を最小構成で動かす

ゴール：`http://localhost:3000/health` にアクセスすると `{"status":"ok"}` が返る。

### Step 1-1. package.json を作る
```powershell
node -v        # v24.21.0 と出ることを確認（違ったら VS Code を開き直す）
npm init -y
```
できた `package.json` を開き、`"type": "module"` を 1 行足す（ESM で書くため）。
```json
{
  "name": "express-crud-api",
  "version": "1.0.0",
  "type": "module",
  ...
}
```
✅ 確認：`package.json` ができている

### Step 1-2. Express と TypeScript を入れる
```powershell
npm i express
npm i -D typescript tsx @types/express @types/node@24
```
✅ 確認：`node_modules/` と `package-lock.json` ができている

### Step 1-3. tsconfig.json を作る
```powershell
npx tsc --init
```
雛形ができるので、開いて**次のところを直す**。それ以外はそのままでよい。

| 項目 | 雛形の値 | 直す値 | 理由 |
|---|---|---|---|
| `types` | `[]` | `["node"]` | `process` などの Node の型を使うため |
| `outDir`（コメントアウトされている） | — | `"dist"` を有効にする | ビルド結果の置き場所 |
| `jsx` | `"react-jsx"` | 行ごと削除 | React は使わない |
| `declaration` / `declarationMap` | `true` | 行ごと削除 | ライブラリではないので型定義ファイルは不要（残すと `app` の型でエラーが出ることがある） |
| `exactOptionalPropertyTypes` | `true` | 行ごと削除 | Zod や Prisma の任意項目と相性が悪く、最初は混乱しやすい |

`module: "nodenext"` と `strict: true` は雛形で最初から入っている。

✅ 確認：`npx tsc --noEmit` で `No inputs were found` と出る（まだ `.ts` がないので正常）

### Step 1-4. アプリを書く
`src/app.ts`（アプリの組み立て。**listen はしない**）
```ts
import express from "express";

export const app = express();
app.use(express.json());

app.get("/health", (_req, res) => {
  res.json({ status: "ok" });
});
```

`src/server.ts`（起動するだけ）
```ts
import { app } from "./app.js";

const port = Number(process.env.PORT ?? 3000);
app.listen(port, () => {
  console.log(`http://localhost:${port}`);
});
```
> import のパスは `./app.ts` ではなく **`./app.js`** と書く（`nodenext` のルール）。

### Step 1-5. 起動スクリプトを足して動かす
`package.json` の `"scripts"` に `dev` を足す。
```json
"scripts": {
  "dev": "tsx watch src/server.ts"
}
```
```powershell
npm run dev
```
✅ 確認：別のターミナルで
```powershell
curl.exe http://localhost:3000/health
```
→ `{"status":"ok"}` が返る。`app.ts` を書き換えて保存すると自動で再起動する。

### Step 1-6. コミット
```powershell
git add .
git commit -m "feat: minimal express server"
```

---

## Phase 2：テストを書けるようにする

ゴール：`npm test` で `/health` のテストが通る。

### Step 2-1. Vitest と supertest を入れる
```powershell
npm i -D vitest supertest @types/supertest
```

### Step 2-2. テストを書く
`tests/health.test.ts`
```ts
import request from "supertest";
import { describe, expect, it } from "vitest";
import { app } from "../src/app.js";

describe("GET /health", () => {
  it("200 と status: ok を返す", async () => {
    const res = await request(app).get("/health");
    expect(res.status).toBe(200);
    expect(res.body).toEqual({ status: "ok" });
  });
});
```
> `app.ts` で listen していないので、サーバーを起動しなくてもテストできる。

### Step 2-3. スクリプトを足して実行
```json
"scripts": {
  "dev": "tsx watch src/server.ts",
  "test": "vitest run",
  "test:watch": "vitest"
}
```
```powershell
npm test
```
✅ 確認：`1 passed` と出る。VS Code の左のテスト（フラスコ）アイコンからも実行できる

### Step 2-4. コミット
```powershell
git add .
git commit -m "test: add health check test"
```

---

## Phase 3：DB（Prisma + SQLite）を用意する

ゴール：Todo テーブルができて、Prisma Studio で中身が見える。

### Step 3-1. Prisma を入れて初期化
```powershell
npm i @prisma/client@7 @prisma/adapter-better-sqlite3@7 dotenv
npm i -D prisma@7
npx prisma init --datasource-provider sqlite
```
作られるもの：

| ファイル | 中身 |
|---|---|
| `prisma/schema.prisma` | テーブル定義を書くファイル |
| `prisma7.config.ts` | Prisma CLI の設定（DB の場所など） |
| `.env` | `DATABASE_URL="file:./dev.db"` |
| `.gitignore` | `/generated/prisma` などが追記される |

> ⚠️ `prisma init` は **`.claude/skills/`・`.windsurf/skills/`・`.agents/skills/`・`skills-lock.json`**（AI ツール向けのファイル）も作る。使わないなら削除してよい。

✅ 確認：上のファイルができている

### Step 3-2. Todo モデルを書く
`prisma/schema.prisma` の末尾に足す。
```prisma
model Todo {
  id        Int       @id @default(autoincrement())
  title     String
  done      Boolean   @default(false)
  dueDate   DateTime?
  createdAt DateTime  @default(now())
  updatedAt DateTime  @updatedAt
}
```

### Step 3-3. マイグレーションしてクライアントを作る
```powershell
npx prisma migrate dev --name init_todo
npx prisma generate
```
> Prisma 7 は `migrate dev` だけではクライアントを作り直さないので、`generate` も実行する。

✅ 確認
- `prisma/migrations/` にフォルダができている
- `generated/prisma/` ができている
- `npx prisma studio` でブラウザが開き、Todo テーブルが見える（または VS Code で `dev.db` をクリック）

### Step 3-4. アプリから使う PrismaClient を作る
`src/lib/prisma.ts`
```ts
import "dotenv/config";
import { PrismaBetterSqlite3 } from "@prisma/adapter-better-sqlite3";
import { PrismaClient } from "../../generated/prisma/client.js";

const adapter = new PrismaBetterSqlite3({ url: process.env.DATABASE_URL! });
export const prisma = new PrismaClient({ adapter });
```
> Prisma 7 は DB につなぐ**アダプタを渡すのが必須**。

### Step 3-5. スクリプトを足してコミット
```json
"db:migrate": "prisma migrate dev",
"db:generate": "prisma generate",
"db:studio": "prisma studio"
```
```powershell
git add .
git commit -m "chore: set up prisma with sqlite"
```
✅ 確認：`git status` で `dev.db` と `.env` がコミット対象に**入っていない**

---

## Phase 4：TODO API を作る

ゴール：TODO の作成・一覧・取得・更新・削除が API でできて、テストが通る。

最初は `routes` のファイルに直接書き、動いてから整理する。

### Step 4-1. テスト用の DB を分ける
テストで開発用の `dev.db` を消さないように、テスト用の DB を使う。

`vitest.config.ts`
```ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    env: { DATABASE_URL: "file:./test.db" },
    globalSetup: "./tests/globalSetup.ts",
    fileParallelism: false,
  },
});
```

`tests/globalSetup.ts`（テスト開始前に一度だけ、テスト用 DB を作り直す）
```ts
import { execSync } from "node:child_process";

export default function setup() {
  execSync("npx prisma migrate reset --force", {
    env: { ...process.env, DATABASE_URL: "file:./test.db" },
    stdio: "inherit",
  });
}
```
✅ 確認：`npm test` で `test.db` が作られ、`/health` のテストが通ったまま

### Step 4-2. POST /todos（作成）
1. **Zod を入れる**
   ```powershell
   npm i zod
   ```
2. `src/modules/todos/todo.schema.ts` に作成用のスキーマを書く（`title` は 1〜100 文字、`dueDate` は任意）
3. `src/modules/todos/todo.routes.ts` に `POST /` を書く
   - `createTodoSchema.parse(req.body)` で検証 → `prisma.todo.create()` → `res.status(201).json(todo)`
4. `app.ts` に `app.use("/todos", todoRouter)` を足す
5. 手で試す：`requests.http`（REST Client）を作って送る
   ```http
   POST http://localhost:3000/todos
   Content-Type: application/json

   { "title": "牛乳を買う" }
   ```
✅ 確認：201 と作成された TODO が返る

### Step 4-3. エラーを JSON で返すようにする
Step 4-2 で `title` を空にして送ると、**500 と HTML のエラー画面**が返る。ここでエラー処理を作る。

1. `src/middlewares/errorHandler.ts` を作る
   - `ZodError` → 400、`{ error: { code: "VALIDATION_ERROR", message, details } }`
   - それ以外 → 500、`{ error: { code: "INTERNAL_ERROR", message } }`
2. `app.ts` の**一番最後**に `app.use(errorHandler)` を置く

> Express 5 では、async ハンドラの中で throw したエラーも自動でここに届く。`try/catch` は要らない。

✅ 確認：`title` を空にして送ると 400 と JSON が返る

### Step 4-4. テストを書く
`tests/todos.test.ts`
- 正しい入力 → 201、`id` がある
- `title` が空 → 400

`tests` の中で毎回 DB を空にする：`beforeEach(() => prisma.todo.deleteMany())`

✅ 確認：`npm test` が通る

### Step 4-5. GET /todos と GET /todos/:id
1. 一覧：`prisma.todo.findMany({ orderBy: { createdAt: "desc" } })`
   - `?done=true` で絞り込めるようにする（`req.query` を Zod で変換）
2. 1 件取得：`id` を `z.coerce.number()` で数値にして `findUnique`
3. 見つからないときのために `src/errors.ts` に `NotFoundError` を作り、`errorHandler` で 404 にする
4. テストを足す
   - 2 件作る → 一覧で 2 件
   - 存在しない id → 404
   - `abc` のような id → 400

✅ 確認：`npm test` が通る

### Step 4-6. PATCH /todos/:id と DELETE /todos/:id
1. 更新：作成用スキーマの `.partial()` に `done` を足したスキーマで検証
2. 削除：成功したら 204
3. どちらも、存在しない id なら 404
4. テストを足す
   - `{ done: true }` で更新できる
   - 削除後に GET すると 404

✅ 確認：`npm test` が通る。`requests.http` から手でも一通り試す

### Step 4-7. 整理する（リファクタリング）
`todo.routes.ts` が長くなってきたら分ける。
- `todo.service.ts`：Prisma を使う処理
- `todo.routes.ts`：リクエストを受けて service を呼ぶだけにする

✅ 確認：**テストが通ったまま**であること（テストがあるので安心して整理できる）

### Step 4-8. 型チェックとコミット
```json
"typecheck": "tsc --noEmit"
```
```powershell
npm run typecheck
npm test
git add .
git commit -m "feat: TODO CRUD API"
git push
```

---

## Phase 5：予約 API を作る

ゴール：部屋ごとに予約ができ、時間が重なる予約は 409 で断られる。

### Step 5-1. モデルを足す
`schema.prisma` に `Room` と `Reservation` を足す（README の「データモデル」参照）。
```powershell
npx prisma migrate dev --name add_reservation
npx prisma generate
```
テストの `beforeEach` に `reservation.deleteMany()` → `room.deleteMany()` を足す（**子テーブルから先に**消す）。

✅ 確認：`npm test` で TODO のテストが通ったまま

### Step 5-2. Room API
TODO と同じ作り方で `POST /rooms`・`GET /rooms`・`GET /rooms/:id` を作る。
- 部屋の名前が重複したら 409（Prisma のエラーコード `P2002` を `ConflictError` にして返す）

### Step 5-3. POST /reservations（一番の山場）
1. スキーマ：`roomId`・`guestName`・`startAt`・`endAt`
   - `.refine` で `startAt < endAt`
   - `.refine` で `startAt` が未来
2. 重なりの判定：同じ部屋に「`既存.startAt < 新.endAt` かつ `既存.endAt > 新.startAt`」の予約があれば 409
3. 判定と作成は `prisma.$transaction` の中で行う
4. テスト
   - 正常 → 201
   - `endAt <= startAt` → 400
   - 過去の日時 → 400
   - 存在しない部屋 → 404
   - 時間が重なる → 409
   - ちょうど隣り合う（前の終わり = 次の始まり）→ 201

✅ 確認：`npm test` が通る

### Step 5-4. 残りの API
- `GET /reservations?roomId=1`、`GET /reservations/:id`、`DELETE /reservations/:id`
- `PATCH /reservations/:id`：日時を変えるときは**自分自身を除いて**重なりを判定する

### Step 5-5. コミット
```powershell
npm run typecheck
npm test
git add .
git commit -m "feat: reservation API"
git push
```

---

## Phase 6：発展（任意）

好きなものから選ぶ。

- [ ] ページング（`?page=1&limit=20`）
- [ ] 同じ検証コードが増えてきたら `validate` ミドルウェアにまとめる
- [ ] OpenAPI を生成して Swagger UI で見る
- [ ] ビルドして動かす（`build: tsc`、`start: node dist/src/server.js`）
- [ ] ESLint / Prettier の設定
- [ ] GitHub Actions で push のたびにテスト

---

## 困ったとき

| 症状 | よくある原因 |
|---|---|
| `node -v` が v20 のまま | VS Code・ターミナルを開き直していない |
| `Cannot find module './app'` | import に `.js` を付けていない（`./app.js`） |
| `Cannot find name 'process'` | `tsconfig.json` の `types` に `"node"` がない |
| `Cannot find module '../../generated/prisma/client.js'` | `npx prisma generate` を実行していない |
| `DATABASE_URL` が undefined | `import "dotenv/config"` を書いていない |
| テストで開発用のデータが消えた | `vitest.config.ts` の `DATABASE_URL` が `test.db` になっていない |
| テストが時々失敗する | `fileParallelism: false` を入れていない |
| バリデーションエラーで 500 と HTML が返る | `errorHandler` がない、または `app.use` の最後に置いていない |
| Prisma 8 の rc が入った | `@7` を付けずにインストールした |
