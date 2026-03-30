---
title: "【日本株スイングトレードAI】懲役5年+口座閉鎖の特大リスクにClaude Codeと多層防御で立ち向かった話"
emoji: "🛡️"
type: "tech"
topics: ["ClaudeCode", "Python", "コンプライアンス", "金融データ", "AIエージェント"]
published: true
status: ready
created_at: "2026-03-30"
sketch_source: "content/sketches/data-source-compliance.md"
adr_source: "029"
cross_media:
  note: "adr029-data-source-compliance"
---

> 約4,300銘柄を毎日自動スクリーニングし、独自の多ファクターモデルでスコアリングする日本株スイングトレードAIを、Claude Codeで構築・運用しています。本記事はそのシステムにおけるコンプライアンス対応の技術実装記録であり、金融規制の法的解釈や投資助言を提供するものではありません。

## はじめに

「noteマネー公式が有料記事を宣伝して炎上してる。金商法違反の疑いと、SBI証券のチャートの商用利用が問題になってるらしい」

このニュースを見た瞬間、手が止まった。自分のシステムのデータフローを確認した。SBI証券のスクリーニングCSV → 独自スコアリング → Xで毎日公開。J-Quants APIのデータもスコアリングに使っている。

**これ、規約違反じゃないか？**

Claude Codeに調査を依頼したところ、出てきたリスクが想像以上に大きかった:

- **金商法の投資助言業無登録**: 5年以下の懲役 or 500万円以下の罰金
- **データソース利用規約違反**: 口座閉鎖・サービス利用停止
- **プラットフォーム規約違反**: アカウント停止・全コンテンツ喪失

「免責文を書いておけば大丈夫」ではなかった。**特大リスクだから、多層防御が必要だった。**

この記事は、Claude Codeとの対話を通じてリスクを洗い出し、同じセッションで5層の多層防御をコードに実装するまでの記録です。

法的リスクの詳細な調査結果（データソース利用規約、金商法の境界整理）は[note記事](https://note.com/quant_swing_jp/n/n9c533f664166)に書いています。本記事は**なぜ多層防御なのか**と、その技術実装にフォーカスします。

---

## なぜ「多層防御」が必要なのか

詳細はnote記事に譲りますが、核心だけ整理します。

**リスクの大きさ**:

| リスク | 最悪のシナリオ | 回復可能性 |
|---|---|---|
| 金商法の投資助言業無登録 | **懲役5年 / 罰金500万円** | 不可逆 |
| SBI証券 規約違反 | **口座閉鎖** → スクリーナー入力データ喪失 | 困難 |
| J-Quants 規約違反 | **アカウント停止** → 決算データ喪失 | Pro契約で復旧可 |
| note 規約違反 | **アカウント停止** → 全記事・SEO資産喪失 | 不可逆 |
| X 著作権侵害 | **アカウント凍結** → フォロワー喪失 | 困難 |

これらが**同時に発生しうる**。1つのスコア投稿が、データソース規約違反 + 金商法リスク + プラットフォーム規約違反の3つを同時にトリガーする。

だから免責文1行で済ませるのではなく、**生成自体をさせない多層防御が必要**だった。WARNING を出して人間に判断を委ねるのではなく、システムとして絶対に実行されない設計にする。

**結論**: スコアをXで公開すること自体が、入力データの利用規約に抵触する可能性がある（各社の利用規約に基づく判断。法的詳細は[note記事](https://note.com/quant_swing_jp/n/n9c533f664166)を参照）。免責文で「独自算出」と書いても、入力データの規約違反は解消されない。

---

## 人間×AIの対話から多層防御が生まれるまで

このセッションでは、人間（自分）とAI（Claude Code）が対話しながら段階的に問題の深さに気づき、対応範囲を拡大していきました。その過程をそのまま記録します。

### Round 1: 「ADRを起票して調査して」

最初の指示は「炎上をきっかけに、データソースの利用規約を調査してADRを起票してほしい」でした。

Claude Codeは2つの調査エージェントを並列起動:
- 1つ目: リポジトリ内のデータフローとコンテンツパイプラインの探索
- 2つ目: 各データソースの利用規約、金商法、プラットフォーム規約のWeb調査

調査結果から ADR-029（8つの決定事項）が起票され、コンプライアンスチェックのコード変更が実装されました。

この時点では「免責文を強化してスコア公開は継続」という方針でした。

### Round 2: 人間「Xの投稿、全体的にダメな気がする」

AIが投稿タイプ別のリスク評価テーブルを出してきた。trending-score（個別銘柄の深堀り）が最もリスクが高く、ほぼ**銘柄分析レポート**だと。

でも自分が気になったのは投資助言の問題ではなかった。

### Round 3: 人間「投資助言もそうですが、スコアの方です」

**ここが転換点でした。**

> 人間: 「スコアを出すこと自体が、入力ソースとの兼ね合いからどうかと思っています」

AIの最初の提案は「免責文を強化して継続」だった。しかし人間が問題の本質を指摘した:**スコアはSBIデータの加工物であり、免責文では入力データの規約違反は解消されない。**

```
SBI スクリーナー CSV（二次利用禁止・加工含む）
  → Phase 1（多ファクタースクリーニング）スコアの主入力
    （return_1d, RSI, SMA乖離...全て SBI 由来）
  → Phase 1 スコア = SBI データの加工物
  → X に公開 = 規約が禁止する「加工データの第三者提供」
```

この指摘を受けて、方針を「スコア公開の停止」に転換。ADR-029にD8（スコア公開停止）を追記しました。

### Round 4: 人間「hookの方が強いのでは？」

`post_to_x.py` に WARNING を追加したところ、人間から「WARNINGは無視して実行できてしまう。hookの方が強い」という指摘。

その通りです。Claude Codeの PreToolUse hook なら**ツール実行自体をブロック**できます。

```bash
#!/bin/bash
# ADR-029 D8: score-ranking / trending-score の実行をブロックする PreToolUse hook

INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // ""')

if echo "$COMMAND" | grep -qE \
  '(--type\s+(score-ranking|trending-score)|--type=(score-ranking|trending-score))'; then
  echo "BLOCK: ADR-029 D8 により score-ranking / trending-score は廃止されました。"
fi
```

settings.json に登録:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/block-score-posting.sh"
          }
        ]
      }
    ]
  }
}
```

### Round 5: 人間「毎回止まるのは困る」

hookで止まるのはセーフティネットとしては正しいが、日次処理（full-daily）で毎回hookに当たって止まるのは運用上困る。

「hookに到達する前に、上流で生成自体をさせないべきでは？」

この指摘から、パイプライン全体を追跡するExploreエージェントを起動。SKILL.md、posting_schedule.json、content-plan、x-post、post_scheduler_cron.py など全ファイルを洗い出し、**5層の多層防御**が設計されました。

---

## 技術実装の詳細

### 1. Policy-as-Code: `data_source_policy.py`

データソースの利用制限をPythonのデータクラスで定義します。

```python
class UsageTier(str, Enum):
    RESTRICTED = "restricted"        # 公開コンテンツへの使用に制約あり
    EXECUTION_ONLY = "execution_only"  # トレード執行のみ
    UNRESTRICTED = "unrestricted"    # 制限なし

@dataclass(frozen=True)
class DataSourcePolicy:
    name: str
    tier: UsageTier
    allows_free_content_derivation: bool
    allows_paid_content_derivation: bool
    allows_raw_data_publication: bool
    allows_screenshot_publication: bool
    upgrade_path: str | None = None

SBI_SCREENER = DataSourcePolicy(
    name="SBI スクリーナー",
    tier=UsageTier.RESTRICTED,
    allows_free_content_derivation=True,
    allows_paid_content_derivation=False,
    allows_raw_data_publication=False,
    allows_screenshot_publication=False,
    upgrade_path=None,
)

JQUANTS_LIGHT = DataSourcePolicy(
    name="J-Quants Light",
    tier=UsageTier.RESTRICTED,
    allows_free_content_derivation=True,
    allows_paid_content_derivation=False,  # Proが必要
    allows_raw_data_publication=False,
    allows_screenshot_publication=False,
    upgrade_path="J-Quants Pro（法人向け別契約）",
)
```

将来J-Quants Proに移行したとき、`allows_paid_content_derivation` を `True` にするだけでマネタイズゲートが解除されます。ポリシー変更がコード変更として追跡可能です。

### 2. コンプライアンスチェックの3つの新カテゴリ

`content_compliance_base.py` に追加した3つのチェック:

#### データソース漏洩チェック

SBIやJ-Quantsの内部用語がコンテンツに漏洩していないか検出します。

```python
_DATA_SOURCE_LEAK_PATTERNS = [
    {
        "pattern": r"SBI(?:証券|スクリーナー)(?:の|から|より)(?:データ|情報|結果)",
        "suggestion": "SBI証券をデータソースとして明示しないでください",
    },
    {
        "pattern": r"J-?Quants(?:の|から|より)(?:データ|情報|決算)",
        "suggestion": "J-Quantsをデータソースとして明示しないでください",
    },
    # ...
]

def check_data_source_leaks(text: str) -> list[ArticleViolation]:
    violations = []
    for leak in _DATA_SOURCE_LEAK_PATTERNS:
        match = re.search(leak["pattern"], text, re.IGNORECASE)
        if match:
            violations.append(ArticleViolation(
                rule="data_source_leak",
                category="data_source",
                matched_text=match.group(),
                severity="block",
                suggestion=leak["suggestion"],
            ))
    return violations
```

#### マネタイズゲート

有料コンテンツに個別銘柄コード+スコアの組み合わせが含まれる場合、自動ブロック。

```python
_TICKER_SCORE_PATTERN = re.compile(
    r"(?:^|[\s　])(\d{4})[\s　].*?(?:スコア|score|評価|ランク|rating).*?[\d.]+",
    re.IGNORECASE | re.MULTILINE,
)

def check_monetization_gate(text: str, is_paid: bool) -> list[ArticleViolation]:
    if not is_paid:
        return []
    if _TICKER_SCORE_PATTERN.search(text):
        return [ArticleViolation(
            rule="monetization_gate",
            category="monetization_gate",
            matched_text="有料コンテンツに銘柄コード+スコアの組み合わせ",
            severity="block",
            suggestion="有料コンテンツで個別銘柄のスコアを公開すると、"
                       "金商法・データソース規約に抵触するリスクがあります（ADR-029 D4）",
        )]
    return []
```

### 3. 5層の多層防御

最終的に、以下の5層で廃止投稿タイプの実行を防いでいます。

```
層1: 指示レベル（SKILL.md, content-plan, x-post）
  ↓ Claude がそもそも score-ranking を生成しようとしない
層2: スケジュール（posting_schedule.json）
  ↓ 日次スケジュールにスコア投稿が存在しない
層3: スクリプト（post_to_x.py: sys.exit(1)）
  ↓ CLI で直接呼んでも即座にエラー終了
層4: cron ガード（post_scheduler_cron.py）
  ↓ DBにドラフトが残っていても投稿されない
層5: hook（PreToolUse: BLOCK）
  ↓ Claude Code 経由の実行を最終ブロック
```

#### 層3: スクリプトレベル

```python
elif args.type == "score-ranking":
    print(
        "ERROR: score-ranking は ADR-029 D8 により廃止されました。"
        "データソース利用規約違反のリスクがあるため実行できません。",
        file=sys.stderr,
    )
    sys.exit(1)
```

#### 層4: cron ガード

```python
_DEPRECATED_TYPES = {"score-ranking", "trending-score"}
if post.post_type in _DEPRECATED_TYPES:
    reason = f"ADR-029 D8: {post.post_type} は廃止済み"
    logger.warning("BLOCK id=%d: %s", post.id, reason)
    ph_repo.mark_failed(session, post.id, reason)
    return False
```

### 4. テスト

26テスト全てPASS。

```python
class TestMonetizationGate:
    def test_paid_article_with_ticker_and_score_is_blocked(self):
        text = "3923 ラクス — スコア 0.87\n投資助言ではありません。"
        violations = check_monetization_gate(text, is_paid=True)
        assert len(violations) == 1
        assert violations[0].severity == "block"

    def test_free_article_with_ticker_and_score_is_ok(self):
        text = "3923 ラクス — スコア 0.87"
        violations = check_monetization_gate(text, is_paid=False)
        assert len(violations) == 0

class TestDataSourcePolicy:
    def test_free_score_publication_allowed(self):
        assert can_publish_derived_score(is_paid=False) is True

    def test_paid_score_publication_blocked(self):
        assert can_publish_derived_score(is_paid=True) is False
```

---

## ADR駆動開発としての位置付け

今回の対応は、1つのセッションで以下が完了しました:

1. **ADR-029 起票**: 調査結果と8つの決定事項
2. **Policy-as-Code**: データソース制限の構造化
3. **コンプライアンスチェック**: 3つの新カテゴリ
4. **免責文更新**: 3ファイル
5. **パイプライン修正**: SKILL.md × 3、posting_schedule.json、ADR-020改訂
6. **5層防御の実装**: スクリプト、cron、hook
7. **テスト**: 26テスト作成・全PASS
8. **ドキュメント**: ADR-029、phase-f-early.md、pipelines/README.md

**人間の判断が実装範囲を決定する構造** が重要です。

AIは最初「免責文を強化して継続」を提案しました。人間が「入力データの規約違反は免責文で解消されない」と指摘して初めて、スコア公開の停止に方針転換。さらに「hookの方が強い」「毎回止まるのは困る」という運用視点の指摘から5層防御が生まれました。

AIは調査と実装を高速で回せますが、**何を守り、何を止めるかの判断は人間がする**。この分業こそが、ADR駆動開発の核です。

---

## まとめ

- 金融データの利用規約違反は**懲役・罰金・口座閉鎖**レベルの特大リスク。免責文1行で守れる規模ではない
- だから多層防御。「指示→スケジュール→スクリプト→cron→hook」の5層で、生成自体をさせない設計にした
- Policy-as-Codeでデータソース制限を構造化し、将来のプラン変更に備える
- 人間×AIの対話で問題の深さに段階的に気づき、対応範囲を拡大する。AIだけでは到達しない判断がある

---

※本記事は筆者の実験記録であり、投資助言ではありません。
