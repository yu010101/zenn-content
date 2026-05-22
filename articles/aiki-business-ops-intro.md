---
title: "aikiを作った — 「人が采配し、AIエージェントが事業運用を回す」AI駆動 事業運用OS"
emoji: "🥋"
type: "tech"
topics: ["claudecode", "ai", "typescript", "mcp", "個人開発"]
published: false
---


## なぜ作ったのか

少人数で複数の事業を回していると、毎日同じことに気づきます。

- KPIの確認、競合のチェック、レポート作成、リードの拾い上げ……「**定常運用**」が人の時間を食い潰す
- かといって全部AIに丸投げすると、**コストが暴走したり、勝手に変なことをしたり**する

結局ほしいのは「全自動」でも「全手動」でもなく、**人が采配（方針と承認）を握り、AIエージェントが定常運用を回す**という分業でした。これを安全レイヤ付きで誰でも再現できる形にしたのが [`aiki`（合気）](https://github.com/AI-Driven-School/aiki) です。

AI駆動開発のツールはコード生成やクリエイティブ生成に寄りがちですが、aikiが狙うのはその一段先、**「事業のオペレーションそのものの自律化」**です。

## aikiとは

`Claude Code` × `MCP` × `GitHub Actions` の上で、役割を持ったエージェント群をスケジュール実行し、**コスト上限・承認ゲート・監査ログ**で安全に回す最小フレームワークです。

| 概念 | 役割 |
|---|---|
| Agent | 役割（KPI監視 / レポート / 分析…）を持つ実行単位 |
| Schedule | いつ回すか（cron / GitHub Actions） |
| SafetyGate | コスト上限・サーキットブレーカー・人間承認で暴走を遮断 |
| Observability | 実行ログ・コスト・成否を記録、乖離で停止 |

## 事業運用を「宣言」する

aikiでは、運用を宣言的に書きます。

```ts
// examples/ops.config.ts
import { defineOps } from "../src/core";
import { kpiWatch, baseballAnalytics } from "../src/agents";

export default defineOps({
  budgetUsd: 1.0, // 1サイクルのコスト上限（超過で即停止）
  agents: [
    { ...kpiWatch,          schedule: "0 9 * * *" },  // 毎朝9時
    { ...baseballAnalytics, schedule: "0 10 * * 1" }, // 毎週月曜（showcase）
  ],
});
```

`npm run ops` で1サイクル走ります。

```
┌─────────┬──────────────────────┬──────┬──────────────────────────────┐
│ agent                │ ok   │ output                                │
├──────────────────────┼──────┼──────────────────────────────┤
│ kpi-watch            │ true │ MRR 1,234,567 → 達成                  │
│ baseball-analytics   │ true │ OPSランキング: Ohtani-like:1.065 ...   │
└──────────────────────┴──────┴──────────────────────────────┘
cost: $0.015 / budget $1.00
```

## いちばん作りたかったのは「安全ゲート」

AIに定常運用を任せるとき、怖いのは品質よりも**コストの暴走**と**やらかし**です。実際、過去に有料APIの呼びすぎで数十万円飛ばした経験があり、これは仕組みで止めるしかないと痛感しました。

なのでaikiは、コストとリスクを**ランタイム側で物理的に止める**ことを最優先にしています。

```ts
// src/safety.ts（抜粋）
export class Safety {
  private spent = 0;
  private fails = 0;
  constructor(private budgetUsd: number, private maxFails = 3, private requireApproval = false) {}

  charge(usd: number): void {
    this.spent += usd;
    if (this.spent > this.budgetUsd) {
      throw new Error(`budget exceeded: $${this.spent.toFixed(3)} > $${this.budgetUsd.toFixed(3)}`);
    }
  }
  canProceed(): boolean { return this.fails < this.maxFails && this.spent <= this.budgetUsd; }
  approve(action: string): void {
    if (this.requireApproval) throw new Error(`approval required for: ${action}`);
  }
}
```

- **コスト上限**: 1サイクルの予算を超えたら例外を投げて停止
- **サーキットブレーカー**: 失敗が一定数を超えたら以降のエージェントをスキップ
- **承認ゲート**: 破壊的・高コストな操作は人間の承認待ちで止める

そしてエージェントが1体コケても**パイプライン全体は止めない**（失敗隔離）。1つの不安定なツールが全体を巻き込まないようにしています。

```ts
// src/core.ts（抜粋）
for (const a of cfg.agents) {
  if (!safety.canProceed()) { /* circuit open: skip */ continue; }
  try {
    const r = await a.run({ state, charge: (usd) => safety.charge(usd) });
    results.push(r);
    safety.record(r.ok);
  } catch (e) {
    results.push({ agent: a.name, output: String(e), ok: false }); // failure isolation
    safety.record(false);
  }
}
```

## ショーケース：野球データ分析エージェント

フレームワークは、面白いデモがあると一気に伝わります。個人的に野球が好きなので、sabermetricsの **OPS（= OBP + SLG）** を計算してランキングするエージェントを同梱しました。

```ts
export function ops(b: Batter): number {
  const obp = (b.h + b.bb) / (b.ab + b.bb);
  const slg = b.tb / b.ab;
  return obp + slg;
}
```

「事業のKPI監視」と「野球の指標分析」が同じランタイム・同じ安全ゲートの上で動く——aikiが*運用の器*であることが、これで伝わると思います。

> 念のため：これは公開データ/サンプルの**分析・可視化**用途です。賭けやオッズ予想は対象外です。

## 設計判断

- **Claude Code(CLI)中核**: IDE依存を避け、CI・サブエージェント・hooksに組み込みやすい
- **MCP**: AI ↔ 自社機能の接続を標準に乗せる（独自プロトコルを作らない）
- **GitHub Actions**: スケジューラを自前で持たず、cronと監査ログをそのまま使う

## 現状（v0.1・正直版）

動く骨格まで来ました。

- ✅ 宣言的ops / 役割エージェント / リトライ・失敗隔離 / コスト上限 / 監査ログ / Actions実行 / 野球デモ
- 🚧 これから: MCPツールの標準アダプタ、Claude Code連携の薄いラッパ、コスト実測連携、承認UI

「本番運用しています」とはまだ言いません（コード完成 ≠ 本番運用）。実適用が回ったら、その数字で続編を書きます。

---

aiki は AI駆動塾（[AI-Driven-School](https://github.com/AI-Driven-School)）の一環として育てていきます。スター・Issue・「こういう運用を載せたい」歓迎です。

📣 AI駆動 事業運用・AIエージェント基盤の導入相談を **週3〜 / 平日日中 / フルリモート** で受けています。気軽に [GitHub](https://github.com/yu010101) まで。
