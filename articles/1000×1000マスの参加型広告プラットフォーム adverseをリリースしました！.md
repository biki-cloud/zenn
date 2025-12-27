---
title: "1000×1000マスの参加型広告プラットフォーム adverseをリリースしました！"
emoji: "🌍"
type: "tech"
topics: ["nextjs", "cloudflare", "canvas", "drizzle", "typescript"]
published: true
aliases:
  - "adverse-platform"
  - "1000x1000-ad-platform"
---

# はじめに

この記事では、**1000×1000 マス（合計 100 万マス）の巨大グリッド**上に広告を配置できる参加型プラットフォーム「AdVerse」を開発した際の技術的な知見を共有します。

https://adverse.pages.dev/home

Canvas API を使った大規模グリッドの描画、Cloudflare D1 と Drizzle ORM を使ったデータベース設計、エッジランタイムでの制約とその対処法など、実践的な内容をまとめました。

## プロジェクト概要

AdVerse は、ユーザーが 1 マスずつ広告を配置できる参加型プラットフォームです。主な機能は以下の通りです：

- **巨大グリッド**: 1000×1000 マスの広告スペース
- **インタラクティブな操作**: ドラッグで移動、ホイールでズーム、クリックで選択
- **リアルタイム追跡**: 広告のクリック数と閲覧数をリアルタイムで追跡
- **範囲クエリ最適化**: ビューポート内のセルのみを効率的に取得

# 技術スタック選定の理由

## Frontend: Next.js 15 + App Router + Canvas API

```typescript
// Canvas APIを使ったグリッド描画の基本構造
const canvas = canvasRef.current;
const ctx = canvas.getContext("2d");

// ビューポート内のセルのみを描画
for (let gridX = minX; gridX <= maxX; gridX++) {
  for (let gridY = minY; gridY <= maxY; gridY++) {
    const pixelX = gridX * cellSize + viewportPixel.x;
    const pixelY = gridY * cellSize + viewportPixel.y;

    // 画面外のセルはスキップ
    if (pixelX + cellSize < 0 || pixelX > canvasWidth) continue;

    ctx.fillStyle = cellData?.ad?.color || "#f3f4f6";
    ctx.fillRect(pixelX, pixelY, cellSize - 1, cellSize - 1);
  }
}
```

**選定理由**:

- Canvas API は DOM 要素を大量に生成せずに描画できるため、パフォーマンスが良い
- Next.js 15 の App Router はエッジランタイムとの相性が良い
- サーバーコンポーネントとクライアントコンポーネントを適切に分離できる

## Backend: Cloudflare D1 + Drizzle ORM

**選定理由**:

- **コスト**: Cloudflare D1 は無料枠が大きく、スケールしやすい
- **パフォーマンス**: エッジで実行されるため、レイテンシが低い
- **型安全性**: Drizzle ORM は TypeScript ファーストで、型安全なクエリが書ける
- **マイグレーション**: Drizzle Kit でスキーマ変更を管理しやすい

# Canvas API を使った大規模グリッドの描画最適化

## 1. ビューポートベースの描画

100 万マスすべてを描画するのは非現実的なため、**ビューポート内のセルのみを描画**します。

```typescript
// ビューポート内のグリッド範囲を計算
const getViewportGridBounds = useCallback(() => {
  const minX = Math.max(0, Math.floor(-viewportPixel.x / cellSize));
  const maxX = Math.min(
    gridSize - 1,
    Math.ceil((canvasWidth - viewportPixel.x) / cellSize)
  );
  const minY = Math.max(0, Math.floor(-viewportPixel.y / cellSize));
  const maxY = Math.min(
    gridSize - 1,
    Math.ceil((canvasHeight - viewportPixel.y) / cellSize)
  );
  return { minX, maxX, minY, maxY };
}, [viewportPixel, cellSize, canvasWidth, canvasHeight, gridSize]);
```

## 2. バッファを追加したスムーズなスクロール

ビューポートの境界ギリギリでデータを取得すると、スクロール時にちらつきが発生します。**バッファを追加**して、少し余裕を持たせます。

```typescript
// バッファを追加してスムーズなスクロールを実現
const buffer = 5;
const fetchMinX = Math.max(0, minX - buffer);
const fetchMaxX = Math.min(gridSize - 1, maxX + buffer);
const fetchMinY = Math.max(0, minY - buffer);
const fetchMaxY = Math.min(gridSize - 1, maxY + buffer);

const response = await fetch(
  `/api/grid?minX=${fetchMinX}&maxX=${fetchMaxX}&minY=${fetchMinY}&maxY=${fetchMaxY}`
);
```

## 3. ピクセル座標とグリッド座標の変換

Canvas API はピクセル座標で描画しますが、データはグリッド座標で管理します。**変換関数を用意**して、一貫性を保ちます。

```typescript
// ピクセル座標をグリッド座標に変換
const pixelToGrid = useCallback(
  (pixelX: number, pixelY: number) => {
    const gridX = Math.floor((pixelX - viewportPixel.x) / cellSize);
    const gridY = Math.floor((pixelY - viewportPixel.y) / cellSize);
    return { gridX, gridY };
  },
  [viewportPixel, cellSize]
);
```

## 4. ズーム機能の実装

マウスホイールでズームする際、**マウス位置を中心にズーム**することで、自然な操作感を実現します。

```typescript
const handleWheel = (e: React.WheelEvent<HTMLCanvasElement>) => {
  e.preventDefault();

  const rect = canvas.getBoundingClientRect();
  const mouseX = e.clientX - rect.left;
  const mouseY = e.clientY - rect.top;

  // マウス位置のグリッド座標を計算（ズーム前）
  const gridX = (mouseX - viewportPixel.x) / cellSize;
  const gridY = (mouseY - viewportPixel.y) / cellSize;

  // ズーム量を計算
  const zoomFactor = e.deltaY > 0 ? 0.9 : 1.1;
  const newCellSize = Math.max(5, Math.min(100, cellSize * zoomFactor));

  // マウス位置を中心にズーム
  const newViewportPixelX = mouseX - gridX * newCellSize;
  const newViewportPixelY = mouseY - gridY * newCellSize;

  setCellSize(newCellSize);
  setViewportPixel({ x: newViewportPixelX, y: newViewportPixelY });
};
```

# 範囲クエリによるパフォーマンス最適化

## Drizzle ORM での範囲クエリ実装

Drizzle ORM の`gte`と`lte`を使って、**範囲クエリを効率的に実装**します。

```typescript
import { gte, lte, and } from "drizzle-orm";

export async function getGridCells(
  minX?: number,
  maxX?: number,
  minY?: number,
  maxY?: number
) {
  if (
    minX !== undefined &&
    maxX !== undefined &&
    minY !== undefined &&
    maxY !== undefined
  ) {
    // 範囲クエリ
    return await db
      .select()
      .from(gridCellsTable)
      .where(
        and(
          gte(gridCellsTable.x, minX),
          lte(gridCellsTable.x, maxX),
          gte(gridCellsTable.y, minY),
          lte(gridCellsTable.y, maxY)
        )
      );
  }

  // 全て取得（パフォーマンス注意）
  return await db.select().from(gridCellsTable);
}
```

## 広告情報の結合取得

セル情報と広告情報を**効率的に結合**して取得します。

```typescript
// API Routeでの実装
const cells = await getGridCells(
  parseInt(minX),
  parseInt(maxX),
  parseInt(minY),
  parseInt(maxY)
);

// 広告情報も一緒に取得
const cellsWithAds = await Promise.all(
  cells.map(async (cell) => {
    let ad = null;
    if (cell.adId) {
      ad = await getAd(cell.adId);
    }
    return { cell, ad };
  })
);

return Response.json({ cells: cellsWithAds });
```

**注意点**: 現在の実装では N+1 問題が発生する可能性があります。本番環境では、JOIN クエリやバッチ取得を検討すべきです。

# Cloudflare D1 + Drizzle ORM の実装

## スキーマ設計

```typescript
// グリッドセルテーブル（1000x1000のグリッド）
export const gridCellsTable = sqliteTable("grid_cells", {
  cellId: text("cellId").primaryKey(), // "x_y" 形式（例: "100_200"）
  x: integer("x").notNull(),
  y: integer("y").notNull(),
  adId: text("adId"), // 広告ID（nullの場合は空きマス）
  userId: text("userId"), // マスの所有者
  isSpecial: integer("isSpecial", { mode: "boolean" }).notNull().default(false),
  createdAt: integer("createdAt", { mode: "timestamp" })
    .notNull()
    .$defaultFn(() => new Date()),
});

// 広告テーブル
export const advertisementsTable = sqliteTable("advertisements", {
  adId: text("adId").primaryKey(),
  userId: text("userId").notNull(),
  name: text("name"),
  title: text("title"),
  message: text("message"),
  targetUrl: text("targetUrl"),
  color: text("color").notNull().default("#3b82f6"),
  clickCount: integer("clickCount").notNull().default(0),
  viewCount: integer("viewCount").notNull().default(0),
  createdAt: integer("createdAt", { mode: "timestamp" })
    .notNull()
    .$defaultFn(() => new Date()),
});
```

**設計のポイント**:

- `cellId`を`"x_y"`形式の文字列で管理することで、インデックスを効率的に使える
- `x`と`y`にインデックスを張ることで、範囲クエリが高速化される
- 広告情報は別テーブルに分離し、正規化を実現

## Drizzle Kit の設定

Cloudflare D1 と Drizzle Kit を連携させるには、**環境に応じた設定**が必要です。

```typescript
// drizzle.config.ts
export default env.DB_LOCAL_PATH
  ? defineConfig({
      schema: "./src/server/db/schema.ts",
      dialect: "sqlite",
      dbCredentials: {
        url: env.DB_LOCAL_PATH, // ローカル開発用
      },
    })
  : defineConfig({
      schema: "./src/server/db/schema.ts",
      out: "./migrations",
      driver: "d1-http",
      dialect: "sqlite",
      dbCredentials: {
        accountId: env.CLOUDFLARE_ACCOUNT_ID!,
        token: env.CLOUDFLARE_USER_API_TOKEN!,
        databaseId: env.DB_REMOTE_DATABASE_ID!,
      },
    });
```

# エッジランタイムでの制約と対処法

## 1. Edge Runtime の指定

Next.js の API Route でエッジランタイムを使用するには、**明示的に指定**します。

```typescript
export const runtime = "edge";
```

## 2. 制約事項

Cloudflare Pages のエッジランタイムには以下の制約があります：

- **Node.js API の制限**: `fs`や`path`などの Node.js 固有の API は使えない
- **ISR の制限**: `revalidatePath`などの ISR 機能は Node.js ランタイムでのみ動作
- **実行時間の制限**: リクエストの実行時間に制限がある

## 3. 対処法

### データベース接続の最適化

```typescript
// server/db/index.ts
import { drizzle } from "drizzle-orm/d1";
import * as schema from "./schema";

export const db = drizzle(env.DB, { schema });
```

エッジランタイムでは、**D1 データベースへの接続が自動的に最適化**されます。

### エラーハンドリング

```typescript
try {
  const result = await placeAdOnCell(x, y, userId, adData);
  return Response.json({ success: true, ...result });
} catch (error) {
  console.error("Error placing ad:", error);
  const errorMessage =
    error instanceof Error ? error.message : "Failed to place ad";
  return Response.json({ error: errorMessage }, { status: 500 });
}
```

エッジランタイムでは、**エラーメッセージを適切に処理**することが重要です。

# URL パラメータによる状態管理

グリッド上のセル選択状態を**URL パラメータで管理**することで、共有可能なリンクを実現します。

```typescript
// URLパラメータからセルを読み込む
useEffect(() => {
  const xParam = searchParams.get("x");
  const yParam = searchParams.get("y");

  if (xParam && yParam) {
    const x = parseInt(xParam, 10);
    const y = parseInt(yParam, 10);

    if (
      !isNaN(x) &&
      !isNaN(y) &&
      x >= 0 &&
      x < gridSize &&
      y >= 0 &&
      y < gridSize
    ) {
      setSelectedCell({ x, y });

      // セルが画面の中央に来るようにビューポートを調整
      const cellPixelX = x * cellSize;
      const cellPixelY = y * cellSize;
      const targetViewportX = canvasWidth / 2 - cellPixelX;
      const targetViewportY = canvasHeight / 2 - cellPixelY;

      setViewportPixel({
        x: Math.max(-maxX, Math.min(0, targetViewportX)),
        y: Math.max(-maxY, Math.min(0, targetViewportY)),
      });
    }
  }
}, [searchParams, gridSize, cellSize]);

// セルが選択されたときにURLを更新する
const updateSelectedCell = useCallback(
  (cell: { x: number; y: number } | null) => {
    setSelectedCell(cell);

    if (cell) {
      const params = new URLSearchParams(searchParams.toString());
      params.set("x", cell.x.toString());
      params.set("y", cell.y.toString());
      router.replace(`?${params.toString()}`, { scroll: false });
    }
  },
  [searchParams, router]
);
```

# パフォーマンス最適化のポイント

## 1. 描画の最適化

- **画面外のセルは描画しない**: ビューポート外のセルは`continue`でスキップ
- **再描画の最小化**: `useEffect`の依存配列を適切に設定
- **メモ化**: `useCallback`と`useMemo`を活用

## 2. データ取得の最適化

- **範囲クエリ**: 必要な範囲のデータのみを取得
- **バッファの活用**: スクロール時のちらつきを防ぐ
- **キャッシュ**: React Query などのキャッシュライブラリを検討

## 3. 状態管理の最適化

- **Map データ構造**: セルデータは`Map`で管理し、O(1)の検索を実現
- **URL パラメータ**: 状態を URL で管理することで、ブラウザの戻る/進むボタンに対応

# まとめ

1000×1000 マスの巨大グリッドを実現するには、以下の技術が重要でした：

1. **Canvas API**: DOM 要素を大量に生成せずに描画
2. **範囲クエリ**: ビューポート内のデータのみを取得
3. **Cloudflare D1**: エッジで実行される高速なデータベース
4. **Drizzle ORM**: 型安全なクエリビルダー
5. **エッジランタイム**: 低レイテンシな API 実装

これらの技術を組み合わせることで、**スケーラブルでパフォーマンスの良い**アプリケーションを実現できました。

## 今後の改善点

- **N+1 問題の解決**: JOIN クエリやバッチ取得の実装
- **リアルタイム更新**: WebSocket や Server-Sent Events の導入
- **キャッシュ戦略**: React Query などのキャッシュライブラリの導入
- **インデックス最適化**: データベースのインデックス設計の見直し

## 参考リンク

- [AdVerse GitHub Repository](https://github.com/your-username/adverse)
- [Cloudflare D1 Documentation](https://developers.cloudflare.com/d1/)
- [Drizzle ORM Documentation](https://orm.drizzle.team/)
- [Next.js App Router Documentation](https://nextjs.org/docs/app)

---

この記事が、大規模なインタラクティブなアプリケーションを開発する際の参考になれば幸いです。質問やフィードバックがあれば、コメント欄でお知らせください！
