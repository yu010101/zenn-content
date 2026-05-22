---
title: "AIエージェントに「日本人メジャーリーガーのOPSランキング」を毎週出させる — 自作OSS aikiで実データ分析"
emoji: "⚾"
type: "tech"
topics: ["野球", "mlb", "ai", "typescript", "claudecode"]
published: false
---


大谷・村上・鈴木……日本人大リーガーが面白いシーズンです。だったら **AIエージェントに毎週「OPSランキング」を自動で出させればいいのでは？** と思って、自作のAI駆動 事業運用OS [`aiki`（合気）](https://github.com/AI-Driven-School/aiki) の上に作りました。

ポイントは **実名選手 × 実データ**。MLBの公開API（`statsapi.mlb.com`、APIキー不要）から取得するので、数値は本物です。

## 出力（今季・5月時点の実データ）

```
2026 日本人MLB OPS（実データ・OPS降順）
村上宗隆  OPS .934  OBP.382 / SLG.552  HR17  (White Sox)
大谷翔平  OPS .885  OBP.399 / SLG.486  HR 8  (Dodgers)
鈴木誠也  OPS .824  OBP.369 / SLG.455  HR 7  (Cubs)
吉田正尚  OPS .693  OBP.356 / SLG.337  HR 0  (Red Sox)
```

メジャー1年目の**村上宗隆（シカゴ・ホワイトソックス）が、現時点でOPSトップ**。HR17本はインパクトがあります。※打席数がまだ少ない選手（村上172・吉田89打席）も含む途中経過なので、規定到達でまた景色は変わります。

> 補足: 村上はドジャースではなく**ホワイトソックス**移籍です（2025オフにポスティング）。実名で出す以上、所属やデータを盛らない・間違えないのが大前提。

## OPSを30秒で

$$\text{OPS} = \text{OBP（出塁率）} + \text{SLG（長打率）}$$

打率は四球を無視しますが、OPSは**出塁する力と長打する力をまとめた1指標**。「打席でどれだけ貢献したか」をざっくり比べるのに便利です。

## 実装：実データを取りに行くエージェント

MLB公開APIから選手の打撃成績を取り、OPS降順に並べるだけ。person IDを固定しているので、好きな選手に差し替えられます。

```ts
// src/mlb.ts（抜粋）
export const JP_MLB_HITTERS: Record<string, number> = {
  "大谷翔平 (Ohtani)": 660271,
  "村上宗隆 (Murakami)": 808959,
  "鈴木誠也 (Suzuki)": 673548,
  "吉田正尚 (Yoshida)": 807799,
};

export async function fetchHitting(name: string, id: number, season: number) {
  const res = await fetch(
    `https://statsapi.mlb.com/api/v1/people/${id}/stats?stats=season&group=hitting&season=${season}`,
  );
  if (!res.ok) throw new Error(`MLB API ${res.status}`);
  const split = (await res.json())?.stats?.[0]?.splits?.[0];
  if (!split) return null; // そのシーズンに打撃データ無し（投手 / 在籍前）
  const s = split.stat;
  return { name, team: split.team?.name, ops: Number(s.ops), hr: Number(s.homeRuns) /* … */ };
}
```

エージェント側はこれを呼んで並べるだけです。

```ts
export const japaneseMlbRanking: Agent = {
  name: "jp-mlb-ops",
  async run(ctx) {
    ctx.charge(0.01); // コストを安全ゲートに申告
    const season = Number(process.env.AIKI_MLB_SEASON ?? new Date().getFullYear());
    const ranked = await rankByOps(season);
    return { agent: "jp-mlb-ops", output: `${season} 日本人MLB OPS: ...`, ok: ranked.length > 0 };
  },
};
```

## なぜ「aikiの上で」やるのか

ただのスクリプトと違って、aiki に載せると **“運用”が付いてきます**。

```ts
export default defineOps({
  budgetUsd: 1.0,
  agents: [
    { ...kpiWatch,           schedule: "0 9 * * *" },  // 事業のKPI監視
    { ...japaneseMlbRanking, schedule: "0 10 * * 1" }, // 毎週月曜、最新OPSランキング
  ],
});
```

- **毎週自動更新**: GitHub Actions が月曜にランキングを出す（最新スタッツに勝手に追従）
- **安全ゲート**: `ctx.charge()` でコスト申告 → 予算超過で即停止（LLM要約などに拡張してもコスト暴走しない）
- **同居**: 「事業のKPI監視」と「野球のOPS分析」が同じランタイム・同じ安全ルールで動く

つまり aiki は特定用途のツールではなく、**“定期実行する分析・運用の器”**。野球はその器がよく分かるデモです。

## 動かす

```bash
npm i && npm run ops
```
で、上の実データ出力が出ます。シーズンを変えたいときは `AIKI_MLB_SEASON=2025 npm run ops`。

## ここから先

- `JP_MLB_HITTERS` を好きな選手IDに差し替え（投手や他リーグも）
- 通算/月間など期間切り替え、wOBA・OPS+ の追加
- 結果を **LLMで一言コメント化** → Slack通知（「今週は村上が長打で牽引」）
- 順位推移の可視化

> 注: 公開データの**分析・可視化**を目的にしています。賭け・オッズ予想は扱いません。

## まとめ

「好きなもの × AIエージェント × 実データ」は、フレームワークを試す最高の入口でした。野球のOPSランキングも、事業のKPI監視も、aiki から見れば同じ「定期実行する分析エージェント」です。

コード → [AI-Driven-School/aiki](https://github.com/AI-Driven-School/aiki)（スター・「この指標も載せて」歓迎）。フレーム解説の前編は[こちら](https://zenn.dev/yu010101)。

📣 AI駆動 事業運用・AIエージェント基盤の導入相談を **週3〜 / 平日日中 / フルリモート** で受けています → [GitHub](https://github.com/yu010101)
