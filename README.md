# biome-plugins

個人プロジェクトで使っている [Biome](https://biomejs.dev/) 用の GritQL プラグイン集です。
TypeScript + Zod を前提とした、コードベースの一貫性を保つためのカスタムルールが入っています。

## インストール

Biome は `plugins` フィールドにローカルの相対パスしか受け付けないため、利用側のリポジトリには
git submodule で取り込むのが推奨です。

```sh
git submodule add https://github.com/<your-user>/biome-plugins.git biome-plugins
```

`biome.json` で参照します。

```json
{
  "plugins": [
    "./biome-plugins/biome-plugins/no-let.grit",
    "./biome-plugins/biome-plugins/no-new-date.grit"
  ]
}
```

必要なルールだけ列挙してください。すべて `severity="warn"` で報告されます。

## ルール一覧

### 一般 (JS / TS)

#### [`no-let.grit`](./biome-plugins/no-let.grit)
`let` を禁止し `const` を強制します。
**意図:** 再代入が必要なケースは実際にはほとんどなく、可変変数はバグの温床になりやすい。
明示的に必要な場合のみ `// biome-ignore` で許可する運用にする。

#### [`no-new-date.grit`](./biome-plugins/no-new-date.grit)
`new Date()` / `new Date(...)` を禁止します。
**意図:** プロジェクトで dayjs を採用しているため、時刻処理を dayjs に統一する。
テスト時のモック差し替えや TZ 周りの取り回しが一貫する。

#### [`no-nullish-coalescing.grit`](./biome-plugins/no-nullish-coalescing.grit)
`??` 演算子を禁止します。
**意図:** フロントで `??` のフォールバックを書くのは、API レスポンスや型の不備を表面で塗りつぶす行為になりがち。
根本（スキーマや型）を直すべき、というレビュー指針をルール化したもの。

#### [`no-or-fallback.grit`](./biome-plugins/no-or-fallback.grit)
`|| ''` `|| []` `|| {}` `|| null` `|| undefined` のフォールバックを禁止します。
**意図:** `no-nullish-coalescing` と同じ動機。
「値が無いときに空配列/空オブジェクトで誤魔化す」と後段のロジックがバグを隠してしまうので、
欠損は欠損として上に投げる or 例外にする。

#### [`no-type-assertion.grit`](./biome-plugins/no-type-assertion.grit)
`as` 型アサーションを禁止します（`as const` は許可）。
**意図:** `as` は型システムを黙らせるだけで実行時の安全を担保しない。
type guard / discriminator / Zod parse など、検査を伴う方法で narrowing するべき。

#### [`no-while-loop.grit`](./biome-plugins/no-while-loop.grit)
`while` / `do...while` ループを禁止します。
**意図:** 反復処理は `for...of` や配列メソッド（`map` / `filter` / `reduce`）、
あるいは再帰の方が意図が読み取りやすく、無限ループのリスクも減らせる。

---

### Zod スキーマ規約

Zod v4 を前提にしたルール群です。
スキーマの「曖昧さ」を減らし、API の境界で意味が明確になることを優先しています。

#### [`no-bare-z-string.grit`](./biome-plugins/no-bare-z-string.grit)
裸の `z.string()`（および `.nullable()` / `.optional()` / `.default(...)` で終わるもの）を警告。
**意図:** `z.string()` は空文字 `""` を許容する。多くのフィールド（名前・ID・URL 等）では
空文字は実質「欠損」と同じなので、`.nonempty()` `.uuid()` `.url()` 等で意図を明示させる。

#### [`no-tri-state-z-array.grit`](./biome-plugins/no-tri-state-z-array.grit)
`z.array(...).nullable()` / `z.array(...).optional()` を禁止。
**意図:** 「未指定の配列」と「空配列」を区別する意味はほぼ無い。
利用側で `arr ?? []` の判定を強要するのを避け、`.default([])` で空配列を返すか、修飾子を削る。

#### [`no-tri-state-z-boolean.grit`](./biome-plugins/no-tri-state-z-boolean.grit)
`z.boolean().nullable()` / `z.boolean().optional()` を禁止。
**意図:** boolean は true / false の二値であるべき。三状態（true / false / null）を持たせるなら
意味のある enum でモデル化する方が読み手に親切。デフォルトで欠損 = false で良ければ `.default(false)`。

#### [`prefer-z-nonempty.grit`](./biome-plugins/prefer-z-nonempty.grit)
`z.string().min(1)` を `z.string().nonempty()` に置き換えるよう促す。
**意図:** `.min(1)` は値を見ないと意図が読めない。`.nonempty()` の方が「空文字を禁止する」という
意図が明確で、grep もしやすい。

#### [`prefer-z-safe-parse.grit`](./biome-plugins/prefer-z-safe-parse.grit)
Zod スキーマに対する `.parse(...)` / `.parseAsync(...)` を警告し、`safeParse` 系を促す。
**意図:** `.parse()` は失敗時に throw するため、エラーハンドリングが暗黙的になる。
`safeParse` を使って `result.success` で分岐する方が型付きエラーとして扱え、
呼び出し側でハンドリングを忘れにくい。
（`Schema` で終わる変数名、または `z` を含む式に対してのみ発火する。）

#### [`prefer-z-url.grit`](./biome-plugins/prefer-z-url.grit)
`z.string().url()` を `z.url()` に置き換える。
**意図:** Zod v4 でトップレベル builder が追加されたので、短くて意図が直接的な方を使う。

#### [`prefer-z-uuid.grit`](./biome-plugins/prefer-z-uuid.grit)
`z.string().uuid()` を `z.uuid()` に置き換える。
**意図:** `prefer-z-url` と同じく v4 builder への揃え。

## 注意事項

- すべて `language js` で書かれていますが、Biome の GritQL は TypeScript ファイルにも適用されます。
- すべて `severity="warn"`。`biome.json` 側で上書きはできないので、エラーにしたい場合はプラグイン側を編集してください。
- これらは個人プロジェクトの設計思想に強く依存したルールです。汎用ルールではないので、
  採用する場合は意図を読んで合うものだけ選んでください。

## ライセンス

[MIT](./LICENSE)
