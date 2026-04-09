# CLAUDE.md

このファイルは Claude Code (claude.ai/code) がこのリポジトリで作業する際のガイドです。

## コマンド

```bash
# 開発
npm run dev          # ローカル開発サーバー起動
npm run build        # 本番ビルド
npm run build:cached # キャッシュ済み Notion データでビルド（高速）
npm run preview      # 本番ビルドのプレビュー

# コード品質
npm run lint         # ESLint（.js/.ts/.astro）
npm run format       # Prettier フォーマット

# Notion データキャッシュ
npm run cache:fetch  # Notion ブログデータを tmp/ にキャッシュ
npm run cache:purge  # 全キャッシュ削除（nx + tmp/）
```

テストは定義されていない。

## アーキテクチャ

**Notion をデータソースとした Astro 製ブログ**。Notion データベースから記事を取得し、静的サイトを生成する。

### データフロー

1. ビルド時に `@notionhq/client` で Notion データベースから記事を取得（`Published` チェックボックスと `Date <= now` でフィルタ）
2. ブロック内容を再帰的に取得し `tmp/{blockId}.json` にキャッシュ
3. 画像をローカルにダウンロードし、`sharp` で EXIF 除去・リサイズ
4. Astro が静的 HTML を生成。インタラクティブな要素は vanilla JS（`<script is:inline>`）で実装
5. API ルート（`/api/likes.json`）は Vercel 上でサーバーサイド実行

### 主要ファイル

- `src/server-constants.ts` — 環境変数・サイト全体の定数（1ページあたりの記事数、タイムアウト等）
- `src/lib/notion/client.ts` — Notion API 統合の全体：記事取得、ブロック取得、いいね取得・インクリメント
- `src/lib/notion/interfaces.ts` — Notion データ構造の TypeScript 型定義
- `src/layouts/Layout.astro` — ルート HTML レイアウト（OG メタ、サイドバー slot、GA）
- `src/pages/posts/[slug].astro` — 記事詳細ページ。`getStaticPaths()` で SSG
- `src/pages/api/likes.json.ts` — いいね数の GET/POST API（`prerender = false`）

### レンダリング戦略

- `output: 'server'` + Vercel アダプター — デフォルトは SSR
- 記事ページは `export const prerender = true`（`getStaticPaths`）で静的生成
- API ルート（`/api/`）は常にサーバーサイドレンダリング

### Notion ブロック描画

各 Notion ブロックタイプに対応するコンポーネントが `src/components/notion-blocks/` にある。テキスト装飾（太字、斜体、コード等）は `src/components/notion-blocks/annotations/`。トップレベルのディスパッチャーは `src/components/NotionBlocks.astro`。

### いいね機能

**注意: `getAllPosts()` はメモリキャッシュ（`postsCache`）を使うため、Like 値が古くなる。** いいね関連の API では Notion API を直接呼び出してキャッシュをバイパスする必要がある。

- `getLatestLikes(post)` — Notion API（`pages.retrieve`）から最新の Like 値を直接取得（キャッシュバイパス）
- `incrementLikes(post)` — Notion API から最新値を取得してからインクリメント（キャッシュバイパス）

`src/components/LikeButton.astro` は `<script is:inline>` による vanilla JS コンポーネント:

- `localStorage` の `liked:{slug}` で二重いいね防止
- GET `/api/likes.json?slug=` で最新カウント取得（Notion API 直接呼び出し）
- POST `/api/likes.json?slug=` でいいね送信（Notion API 直接呼び出し）
- 楽観的 UI（即座にカウント+1、失敗時ロールバック）

### 環境変数

| 変数                 | 必須 | 説明                              |
| -------------------- | ---- | --------------------------------- |
| `NOTION_API_SECRET`  | Yes  | Notion インテグレーショントークン |
| `DATABASE_ID`        | Yes  | Notion データベース ID            |
| `CUSTOM_DOMAIN`      | No   | 例: `example.com`                 |
| `BASE_PATH`          | No   | サブディレクトリパス              |
| `REQUEST_TIMEOUT_MS` | No   | デフォルト: 10000ms               |

### カスタム Astro ビルドインテグレーション（astro.config.mjs）

ビルド時に実行される4つのカスタムインテグレーション:

- `CoverImageDownloader` — データベースのカバー画像をダウンロード
- `FeaturedImageDownloader` — 全記事のアイキャッチ画像をダウンロード
- `CustomIconDownloader` — カスタム記事アイコンをダウンロード
- `PublicNotionCopier` — ダウンロードしたアセットを出力ディレクトリにコピー
