---
title: "AIエージェントの『コスト暴走』と『やらかし』を止める安全ゲート設計 — 数十万円飛ばした後に作った話"
emoji: "🛡️"
type: "tech"
topics: ["ai", "claudecode", "typescript", "llm", "設計"]
published: false
---


AIエージェントに「定常業務を自律で回す」ことを任せ始めて、痛感したことがあります。**怖いのは出力品質より、「コストの暴走」と「破壊的なやらかし」**だということ。

正直に書くと、過去に有料APIの呼びすぎで**数十万円**を飛ばしました。原因は「想定より多く・速くAPIを叩いてしまった」だけ。精神論や「気をつける」では防げません。**仕組み（ランタイム）で物理的に止める**しかない。

その反省から、自作のAI駆動 事業運用OS [`aiki`](https://github.com/AI-Driven-School/aiki) には**安全ゲートを最優先で組み込みました**。この記事はその設計の話です。

## 前提：自律実行の失敗モードを先に並べる

「速く回す」と「安全に止める」は対立します。これを設計でアウフヘーベンするには、まず**何が壊れるか**を列挙するのが先です。

| 失敗モード | 例 | 対策 |
|---|---|---|
| コスト暴走 | ループでAPI連打 / 想定外の大量処理 | **コスト上限**（超過で停止） |
| 連鎖失敗 | 外部障害で全エージェントが失敗し続ける | **サーキットブレーカー** |
| 破壊的操作 | 本番削除・課金・外部送信 | **人間承認ゲート** |
| 単一障害の波及 | 1エージェントのクラッシュで全停止 | **失敗隔離** |
| 一時的エラー | 瞬断・レート制限 | **リトライ＋タイムアウト＋バックオフ** |
| 不可観測 | 何が起きたか追えない | **監査ログ／コスト記録** |

「LLMは確率的、ガードレールは決定的」。賢いプロンプトで“お願い”するのではなく、**コード側で確定的に止める**のが基本方針です。

## ① コスト上限（申告制で、超えたら例外で止める）

エージェントは実行前に「これくらいコストがかかる」と**申告**し、累積が予算を超えたら例外を投げて止めます。

```ts
// src/safety.ts
export class Safety {
  private spent = 0;
  private fails = 0;
  constructor(
    private readonly budgetUsd: number,
    private readonly maxFails = 3,
    private readonly requireApproval = false,
  ) {}

  charge(usd: number): void {
    this.spent += usd;
    if (this.spent > this.budgetUsd) {
      throw new Error(`budget exceeded: $${this.spent.toFixed(3)} > $${this.budgetUsd.toFixed(3)}`);
    }
  }
  // …
}
```

ポイントは**申告制**にしたこと。各エージェントが `ctx.charge(0.01)` のように自分のコストを申告するので、新しいエージェントを足しても予算管理が一元化されます。理想は実測トークンとの連携ですが、まずは「上限で必ず止まる」ことを優先しています（コード完成 ≠ 本番運用、なので推定→実測はロードマップ）。

> 高コストな一括処理は **dry-run で件数と想定コストを出してから実行** に倒すのも同じ思想。「気づいたら課金されてた」を構造的に消す。

## ② サーキットブレーカー（連鎖失敗で止める）

外部APIが落ちているのに全エージェントを回し続けると、失敗とコストだけが積み上がります。一定回数失敗したら、以降の実行をスキップします。

```ts
  record(ok: boolean): void {
    if (!ok) this.fails++;
  }
  canProceed(): boolean {
    return this.fails < this.maxFails && this.spent <= this.budgetUsd;
  }
```

## ③ 人間承認ゲート（破壊的・高コスト操作は止めて待つ）

本番データの削除、外部送信、課金発生——こういう「取り返しのつかない操作」は、自動で通さない。

```ts
  approve(action: string): void {
    if (this.requireApproval) {
      throw new Error(`approval required for: ${action}`);
    }
  }
```

`AIKI_REQUIRE_APPROVAL=true` のときは破壊的操作で止まり、人間の采配（承認）を待ちます。CI/本番では既定でON、検証では一時的にOFF、と運用で切り替えます。

## ④ 失敗隔離（1体のクラッシュで全体を止めない）

ランタイムは、1つのエージェントが例外を投げても**パイプライン全体は続行**します。これが無いと、不安定な1ツールが全業務を巻き込みます。

```ts
// src/core.ts
for (const a of cfg.agents) {
  if (!safety.canProceed()) { /* circuit open → skip */ continue; }
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

## ⑤ リトライ／タイムアウト／バックオフ（一時障害を吸収）

外部依存は必ず一時的に失敗します。握りつぶさず、有限回リトライしてダメなら諦める。

```ts
export async function callTool(tool, input, { retries = 2, timeoutMs = 3000 } = {}) {
  let lastErr: unknown;
  for (let attempt = 0; attempt <= retries; attempt++) {
    try {
      return await withTimeout(tool(input), timeoutMs);
    } catch (e) {
      lastErr = e;
      await sleep(100 * (attempt + 1)); // backoff
    }
  }
  throw new Error(`tool failed after ${retries + 1} attempts: ${String(lastErr)}`);
}
```

`timeout` を必ず付けるのが地味に効きます。AI/外部呼び出しは「返ってこない」が普通に起きるので、無限待ちを設計から消す。

## ⑥ 観測（何が起きたかを必ず残す）

1サイクルの結果・コスト・状態遷移をログに残します。乖離が見えれば止められる。

```
cost: $0.015 / budget $1.00
state: { kpi: 'MRR 1,234,567 / target 1,000,000', jp_mlb_top: '村上宗隆 OPS 0.934' }
```

## なぜここまでやるのか

AI駆動開発を**自社の遊び**で終わらせず、**他社の本番**に入れるなら、この層が無いと話になりません。「AIに任せました」ではなく「**暴走しても上限で止まり、破壊的操作は承認を挟み、何が起きたか追える**」と言えること。これが、自律性を“安全に”売れる条件だと思っています。

## 限界（v0.1・正直版）

- 推定コスト → 実測トークン/料金の連携は未実装
- 承認は例外停止まで（Web/Slack 承認UIはこれから）
- per-agent 予算、レート制御の細分化もこれから

骨格は動いていますが、「本番運用しています」とはまだ言いません。実適用が回ったら数字付きで続編を書きます。

---

コード → [AI-Driven-School/aiki](https://github.com/AI-Driven-School/aiki)。aiki の全体像（AI駆動 事業運用OS）の話は[前編](https://zenn.dev/yu010101)、実データの野球デモは[こちら](https://zenn.dev/yu010101)。

📣 AI駆動開発の導入・AIエージェント基盤の設計（特に**安全・コストガバナンス**）の相談を **週3〜 / 平日日中 / フルリモート** で受けています → [GitHub](https://github.com/yu010101)
