# AGENTS.md

## プロジェクト

Notion データベースを記事の管理元とする Astro 製ブログ。記事一覧、記事詳細、タグ、RSS を提供する。変更時はブログの読みやすさ、既存の記事表示、モバイル表示を維持する。

## 開発・検証

- `npm run dev`：開発サーバーを起動する。
- `npm run lint`：`src` の JavaScript、TypeScript、Astro を検査する。
- `npm run build`：本番ビルドを実行する。
- `npm run build:cached`：Notion のブロックキャッシュを取得してからビルドする。Notion API への接続が必要。
- `npm run preview`：ビルド結果をプレビューする。
- `npm run cache:fetch`：Notion の記事ブロックを `tmp/` にキャッシュする。
- `npm run cache:purge`：Nx のキャッシュと `tmp/` 内のファイルを削除する。
- `npm run format`：リポジトリ全体を Prettier で書き換える。

自動テストのスクリプトは定義されていない。変更内容に応じて lint、ビルド、画面確認を行い、実行できなかった検証と理由を報告する。Notion からのデータ取得には `NOTION_API_SECRET` と `DATABASE_ID` が必要。

## 実装上の注意

- `astro.config.mjs` は Vercel アダプターと `output: 'server'` を使用する。トップ、記事、ページネーション、タグの各ページはプリレンダリングされ、いいね API はサーバー側で実行される。
- `src/lib/notion/client.ts` の `getAllPosts()` は記事一覧をメモリキャッシュする。最新のいいね数が必要な場合は、Notion API から直接取得する `getLatestLikes()` と `incrementLikes()` の仕組みを維持する。
- Notion ブロックは `src/components/NotionBlocks.astro` と `src/components/notion-blocks/` で描画する。ブロックの変更時は既存の記事への影響も確認する。
- ビルド時の画像取得・コピーは `src/integrations/` にある。生成物や `tmp/` のキャッシュを手作業で変更する前に、生成元を確認する。
- サブディレクトリでの公開に `BASE_PATH` を使用する。リンクやアセットのパスを変更する際は、この設定を考慮する。

環境変数やセットアップの詳細は `src/server-constants.ts` と README を参照する。
