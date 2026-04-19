---
title: "【日本株スイングトレードAI】Claude Code × Remotion でブランド動画を作る — _meta 横断規約・delayRender 罠・5シーン構成"
slug: "claude-code-remotion-brand-video"
emoji: "🎬"
type: "tech"
topics: ["remotion", "claudecode", "typescript", "react", "googlefonts"]
published: false
status: "ready"
created_at: "2026-04-19"
pair_note_slug: "claude-design-vs-claude-code-brand-video"
sketch_ref: "content/sketches/claude-design-vs-claude-code-brand-video.md"
---

## この記事でわかること

- Claude Code に Remotion を組み込んで、X 固定ツイート用の 30 秒ブランド動画を作る具体手順
- 動画プロジェクトをソースリポと物理分離するための `_meta/setup/remotion-video-projects.md` 横断規約の設計
- `Series` + `Series.Sequence` で 5 シーンを「frame 0 から始まる独立シーン」として合成するパターン
- Remotion で日本語フォントを正しく出すための `delayRender` + `waitUntilDone` + `loadFont().fontFamily` の三点セット
- WSL の `Noto Sans CJK JP` フォールバック罠と、`npx remotion still` を「ユニットテスト」として使うデバッグ術
- なぜ Claude Design からの jsx 出力ではなく、Remotion でゼロから書き直す方が結果的に速かったか

---

> 本記事は「Quant Swing」というアカウント（リポ名: `jp-stock-swing-signals`）の自作 AI システムの一部、ブランディング系成果物の制作記録です。Quant Swing は日本株約 4,300 銘柄を毎日スコアリングし、Claude Code の 16 エージェントが「推奨 → 人間承認 → 振り返り → 改善」の OODA ループを回す実験的システムです。本記事は実装メモであり、システムの投資成績や有効性を保証するものではありません。

---

> 実物の動画はこちらに置きました（X 固定ツイート）。記事の前提として 1 度だけ見てもらえると、以降の話が映像と紐付いて読めます。
>
> https://x.com/quant_swing_jp/status/2045801870355259717

---

設計の背景や「なぜ Claude Design ではなく Claude Code に寄せたか」というストーリー側の判断は、note にまとめました。

https://note.com/quant_swing_jp/n/n64101be1bec3

本記事は実装側を書きます。

## ゴールと制約

- **ゴール**: X 固定ツイートに置く 30 秒のブランドイントロ動画 (1080×1080, 30fps, mp4)
- **制約**:
  - ブランドカラーは既存の `app/static/css/base.css` のトークンと完全一致させる
  - 日本語フォント (Noto Sans JP) が正しく表示される
  - ソースリポ (`jp-stock-swing-signals`) には Remotion を入れない（後述の理由）
- **使うもの**: Claude Code + Remotion + `@remotion/google-fonts` + `_meta/setup/remotion-video-projects.md` という横断規約

## 1. 横断規約: `_meta/setup/remotion-video-projects.md` を最初に書く

実装に入る前に、配置規約を `_meta`（複数リポ横断のドキュメント置き場）に切り出しました。

### なぜソースリポに入れないのか

```
D:\Projects\<repo-name>\               ← ソースリポ
D:\Movie\<source-repo-name>\<video-name>\  ← 動画プロジェクト
D:\Movie\<source-repo-name>\<video-name>\out\  ← 最終 mp4
```

理由は 3 つです。

1. **Chromium 同梱**: Remotion はレンダリング時に Chromium を起動します。`@remotion/cli` + `@remotion/renderer` の依存だけで `node_modules` が +400MB 程度に膨らみます。Python メインのソースリポに混ぜると、`uv sync` 後の作業ディレクトリが肥大して GC や CI を圧迫します
2. **CI 時間と成果物汚染**: pytest / lint だけで完結すべきソースリポの CI に、Node の依存解決が混ざると遅くなる。Docker イメージのレイヤも汚れます
3. **「コードと成果物の境界」を明確にする**: 動画は「ソースコード」ではなく「成果物から派生した可視化資産」です。git で追わない。完成動画は X / note / YouTube などのホスティング側にだけ置く

規約の本文（抜粋）:

```markdown
## 配置ルール
| 種類 | 場所 |
|------|------|
| ソースリポ（コード） | `D:\Projects\<repo-name>\` |
| 動画プロジェクト | `D:\Movie\<source-repo-name>\<video-name>\` |
| 最終レンダリング（mp4） | `D:\Movie\<source-repo-name>\<video-name>\out\` |

- Git 管理しない: `D:\Movie` 配下はローカル専用
- 命名: `<video-name>` は kebab-case、用途が一目で分かる名前
- 1 video = 1 project
```

「最初の 1 件で規約を書く」のは退屈ですが、2 本目以降の動画（週次レビュー、ADR 解説）で同じパターンを再利用できるので、トータルでは時間の節約になります。

## 2. Claude Code への Remotion 組み込み（ユーザースコープ）

Claude Code から Remotion を扱うために、**ユーザースコープ**（`~/.claude/`）に Skill と MCP を追加しました。リポスコープではなく `~/.claude/` に置いたのは「全リポで使う」という判断です。

```bash
# Skill: 公式 remotion-dev スキル
ls ~/.claude/skills/remotion/

# MCP: ユーザースコープに登録
# ~/.claude/settings.json などに以下を追加
{
  "mcpServers": {
    "remotion": {
      "command": "npx",
      "args": ["-y", "@remotion/mcp@latest"]
    }
  }
}
```

これで、どのリポから Claude Code を起動しても Remotion 関連の操作（render コマンドの実行支援、コンポーネント雛形の生成、デバッグ補助）が使えます。

## 3. プロジェクトのスキャフォールド

```bash
# 親フォルダ作成（初回のみ）
mkdir -p ~/videos/jp-stock-swing-signals
cd ~/videos/jp-stock-swing-signals

# Remotion プロジェクト作成
npx create-video@latest --yes --blank --no-tailwind x-pinned-brand-intro
cd x-pinned-brand-intro
```

`--blank` を選んだ理由: テンプレが入ると、不要な `Composition` や placeholder 素材を削除する手間の方が、ゼロから書く時間より大きくなります。最小構成から 5 シーンを書き起こす方が早い。

`--no-tailwind` も同じ思想です。CSS 変数を inline style で扱うので Tailwind は要りません。

## 4. 5 シーン合成: `Series` + `Series.Sequence`

`src/Composition.tsx` で 5 シーンを順番に合成します。

```tsx
import { Series } from "remotion";
import { IntroScene } from "./scenes/IntroScene";
import { StatsScene } from "./scenes/StatsScene";
import { LoopScene } from "./scenes/LoopScene";
import { SemanticsScene } from "./scenes/SemanticsScene";
import { OutroScene } from "./scenes/OutroScene";

export const BrandIntroComposition: React.FC = () => {
  return (
    <Series>
      <Series.Sequence durationInFrames={120}>
        <IntroScene />
      </Series.Sequence>
      <Series.Sequence durationInFrames={240}>
        <StatsScene />
      </Series.Sequence>
      <Series.Sequence durationInFrames={240}>
        <LoopScene />
      </Series.Sequence>
      <Series.Sequence durationInFrames={180}>
        <SemanticsScene />
      </Series.Sequence>
      <Series.Sequence durationInFrames={120}>
        <OutroScene />
      </Series.Sequence>
    </Series>
  );
};
```

`Series.Sequence` の重要な性質: **各子コンポーネント内では、`useCurrentFrame()` が「そのシーンの開始 frame=0」から始まる**。つまり `IntroScene` の中では「0〜119 frame」、`StatsScene` の中でも「0〜239 frame」として扱える。シーン間の時間オフセット計算が要らない。

これが地味に効きます。シーン単独でプレビューしたいときも、`Series.Sequence` でラップする側だけを変えれば、シーン本体は触らなくていい。

シーン構成:

| シーン | 時間 | frames | 内容 |
|--------|------|--------|------|
| IntroScene | 0-4s | 120 | "Quant Swing" ワードマーク + サブタイトル + teal アンダーライン sweep |
| StatsScene | 4-12s | 240 | 4,300+ 銘柄/日, 30+ 指標, 16 エージェント を 1 つずつカウントアップ |
| LoopScene | 12-20s | 240 | OODA Loop 4 フェーズを回転する teal pointer で示す |
| SemanticsScene | 20-26s | 180 | 「買い」(teal) + 「売り」(orange) を fontWeight 900 で大きく |
| OutroScene | 26-30s | 120 | @quant_swing_jp + "Built with Claude Code" バッジ |

合計 900 frames @ 30fps = 30 秒。

## 5. デザイントークンの TS 化: `src/tokens.ts`

CSS 変数 (`var(--bg)`) はブラウザランタイムでしか解決されません。Remotion は inline style と `interpolate()` で値を計算で扱うため、ブランドトークンを TypeScript の定数として再定義します。

```ts
// src/tokens.ts
export const colors = {
  bg: "#0f1420",
  surface: "#161e2e",
  text: "#f7f2ea",
  textMuted: "#9ca3b0",
  long: "#2dd4bf",      // 買い建て (現物買い) のセマンティック色
  short: "#fb923c",     // 売り建て (信用売り) のセマンティック色
  accent: "#818cf8",
  divider: "#2a3344",
} as const;

export const spacing = {
  xs: 8,
  sm: 16,
  md: 24,
  lg: 40,
  xl: 64,
} as const;

export const radii = {
  sm: 8,
  md: 16,
  lg: 24,
} as const;
```

参照元の真実は `app/static/css/base.css`（Quant Swing 本体）です。ここを更新したら `tokens.ts` も合わせて更新するだけ。CSS と TS で書き方は違いますが、**値の Single Source of Truth は base.css 側** という運用にすると、ブランドの一貫性が保てます。

## 6. フォントの罠: `Noto Sans JP` ≠ `Noto Sans CJK JP`

ここが**今回一番時間を溶かしたポイント**です。罠の入口と出口を順に書きます。

### 罠の入口: 文字列ハードコード

最初の実装はこうでした。

```tsx
// ❌ NG パターン
<h1 style={{ fontFamily: "Noto Sans JP", fontWeight: "bold" }}>
  買い
</h1>
```

Remotion Studio のプレビューでは「正しく見える」。Chrome のフォントフォールバックが優秀すぎて、`Noto Sans JP` がなければ近いフォントで代用される。視覚的には違和感がない。

ところが `npx remotion render` で書き出すと、書き出された PNG / mp4 では**カーニングが微妙に違う**。文字幅・ウェイト解釈・グリフのバランスがずれる。

原因: WSL のシステム fontconfig が、`Noto Sans JP` を解決できずに **`Noto Sans CJK JP`** にフォールバックしていました。Google Fonts の `Noto Sans JP` と Adobe Source Han Sans 系の `Noto Sans CJK JP` は祖先を共有していますが、別フォント。文字幅・カーニング・ウェイト解釈が違います。

### 罠の出口: `loadFont()` + `delayRender` + `.fontFamily` を export

修正方針は 3 つの要素を組み合わせます。

```tsx
// src/fonts.ts
import { delayRender, continueRender, cancelRender } from "remotion";
import { loadFont as loadNotoSansJP } from "@remotion/google-fonts/NotoSansJP";
import { loadFont as loadUnbounded } from "@remotion/google-fonts/Unbounded";
import { loadFont as loadJetBrainsMono } from "@remotion/google-fonts/JetBrainsMono";

const notoSansJPHandle = loadNotoSansJP("normal", {
  weights: ["400", "500", "700", "900"],
});
const unboundedHandle = loadUnbounded("normal", {
  weights: ["400", "700", "900"],
});
const jetBrainsMonoHandle = loadJetBrainsMono("normal", {
  weights: ["400", "600"],
});

const fontHandle = delayRender("Loading Google Fonts");

Promise.all([
  notoSansJPHandle.waitUntilDone(),
  unboundedHandle.waitUntilDone(),
  jetBrainsMonoHandle.waitUntilDone(),
])
  .then(() => continueRender(fontHandle))
  .catch((err) => cancelRender(err));

// ✅ 解決後の実 family 名を export
export const fontFamilies = {
  notoSansJP: notoSansJPHandle.fontFamily,
  unbounded: unboundedHandle.fontFamily,
  jetBrainsMono: jetBrainsMonoHandle.fontFamily,
};
```

各シーンでの使い方:

```tsx
// src/scenes/SemanticsScene.tsx
import { fontFamilies } from "../fonts";

export const SemanticsScene: React.FC = () => (
  <h1 style={{
    fontFamily: fontFamilies.notoSansJP,  // ✅ ハンドルから取得
    fontWeight: "bold",
  }}>
    買い
  </h1>
);
```

ポイント 3 つ:

1. **`delayRender()` + `waitUntilDone()`**: フォントのダウンロードが完了するまでレンダー全体をブロックする。これがないと「フォント未ロード状態のまま」フレームが書き出される
2. **ハードコード文字列を禁止する**: `loadFont()` が返す `.fontFamily` を export して各シーンで使う。文字列リテラルを書くと、システムフォントへのフォールバックを検出できない
3. **`subsets: ["japanese"]` は使えない**: Noto Sans JP は Unicode range chunk ([0]〜[28]) で配信されており、`japanese` という名前付き subset は存在しません。`subsets` を省略すれば全 range が読み込まれる

### Remotion の `still` コマンドを「ユニットテスト」として使う

フォント問題を 30 秒 × 30fps = 900 frame の動画レンダリングで検証していたら時間が無限にかかります。Remotion には `still`（1 フレームだけ静止画として書き出す）コマンドがあるので、これを使います。

```bash
# 60 frame 目だけ PNG で書き出し
npx remotion still BrandIntro out/frame-60.png --frame=60
```

PNG を目視確認してフォントが想定どおりかをチェックすれば、デバッグループが**約 30 倍速くなります**。

これは Remotion における「ユニットテスト」相当の使い方だと思います。コードの単体テストではなく、視覚出力の単体検証。

## 7. Root に composition を登録

```tsx
// src/Root.tsx
import { Composition } from "remotion";
import { BrandIntroComposition } from "./Composition";
import "./fonts";  // ← フォント初期化のため side-effect で import

export const RemotionRoot: React.FC = () => (
  <Composition
    id="BrandIntro"
    component={BrandIntroComposition}
    durationInFrames={900}
    fps={30}
    width={1080}
    height={1080}
  />
);
```

`import "./fonts"` を side-effect として読み込むことで、`fonts.ts` の `delayRender()` が起動します。これを忘れるとフォントロードが走らず、結局 CJK JP にフォールバックします。

## 8. レンダリング

```bash
npx remotion render BrandIntro out/brand-intro.mp4
```

仕様:

- 解像度: 1080×1080
- フレームレート: 30fps
- 長さ: 900 frames = 30 秒
- 出力: `out/brand-intro.mp4` (2.7MB)

最後にブラウザでフレーム送り確認。Noto Sans JP のグリフが正しく出ていれば完成です。

## 振り返り: なぜ Claude Design からの jsx を採用しなかったか

Claude Design は、リポを読み込ませて専用デザインシステムを生成し、`animations.jsx` を出力できる機能があります。実際に試しました。詳細はストーリー側（note）に書きましたが、技術的な要点だけここに残します。

`animations.jsx` は **React + Babel ベースのブラウザランタイムフレームワーク**でした。`Stage` / `Sprite` / `Timeline` / `useTime()` という独自 DSL を持ち、`Object.assign(window, {...})` でグローバル登録する HTML 前提の構造。`PlaybackBar` で再生・一時停止・シークができます。

ただし、**動画ファイルの書き出し機能は含まれていない**。X 固定ツイートに置くには mp4 が必要なので、

- puppeteer / playwright でブラウザを開いて canvas をフレームキャプチャする、または
- Remotion などのレンダリングパイプラインに DSL を書き直す

のどちらかが追加で必要になります。

一方、Remotion はもともと「React で動画を書く」ことに特化していて、`render` コマンド一発で mp4 が出ます。Claude Code には既にプロジェクトの context が大量に蓄積されており、規約とエビデンスを書きながら実装を進められる。今回のように「最終成果物が mp4」というゴールに対しては、Remotion でゼロから書く方がパス長が短い、という判断でした。

「Claude Design の出力を mp4 化するパイプラインを自作する」よりも、「Remotion でゼロから書く」方が、追加コードの絶対量も、デバッグの不確実性も少なかったです。

## まとめ: 今回の設計判断 7 点

1. **配置規約を最初に書く**: `_meta/setup/remotion-video-projects.md` で `D:\Movie\<source-repo-name>\<video-name>\` を確定
2. **Skill + MCP はユーザースコープ**: 全リポで Remotion を使う前提
3. **`--blank --no-tailwind`**: テンプレ削除より、ゼロから書く方が速い
4. **`Series` + `Series.Sequence`**: 各シーンが「frame 0 開始」になる効率的な合成
5. **トークンを TS 化**: `app/static/css/base.css` を Single Source of Truth とする
6. **フォントは `loadFont().fontFamily` を export**: ハードコード文字列禁止 + `delayRender` + `waitUntilDone` の三点セット
7. **`still` コマンド = ユニットテスト**: 動画書き出しの 30 倍速いデバッグ

特に 6 番のフォント問題は、Web 開発の感覚で `fontFamily: "Noto Sans JP"` と書くと、システムフォントへのフォールバックを目視で検出できないまま本番に行く可能性があるので、Remotion を WSL 環境で使う方は最初から避ける設計をおすすめします。

## まだやっていないこと / 今後

- **動画パイプラインの自動化**: 週次レビュー動画を毎週自動生成したい。可変データ（PnL, トレード件数）を Composition に渡して同じテンプレートで書き出す仕組みは未実装
- **音声**: 今回は無音動画。BGM や TTS ナレーションは未検討
- **`_shared/` コンポーネント**: 横断規約には書いたが、共通コンポーネントの切り出しは 2 本目以降

---

ストーリー側（なぜ Claude Code に寄せたか、Claude Design との比較体験）はこちらに書きました。

https://note.com/quant_swing_jp/n/n64101be1bec3

---

※本記事は筆者の実験記録であり、投資助言ではありません。記載されているスコアリング関連の指標値・スコアは筆者が独自に算出したものであり、その有効性を保証するものではありません。投資判断はご自身の責任でお願いします。
