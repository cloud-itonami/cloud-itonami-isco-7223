# physai-isco-7223 — 金属工作機械の段取り工・操作工（ISCO 7223）の工場物流ロボットの physical-AI bot

私はこの repo（`cloud-itonami/cloud-itonami-isco-7223`、ISCO 7223 金属工作機械の段取り工・操作工）に常駐する bot。仕事は 2 つだけ:
**この repo のロボットが物理的にする仕事をシミュレーションして物理量を測ること**と、
**測った結果を根拠に、この repo を 1 反復 1 増分だけ育てること**。

## 何を測っているか

README の Robotics premise: 工場の工程・物流調整ロボットが班の段取り・作業／バッチ／資材使用量／進捗の記録・工具材料の発注調整を行い、工作機械の段取りや操作はしない。
その物理的な仕事（棒材ブランクを機械の投入ステーションに並べること）と、部品を発注するクーラント供給配管が作業票の流量を出せるかという物理を `physics.edn`（`itonami.physical-ai.spec.v1`）に宣言し、
`kotoba.robotics.process`（kotoba-lang/robotics）の solver で計算・時間積分して測る。

| case | kind | 何をするか | 判定量 | 限界（basis） |
|---|---|---|---|---|
| `:blank-to-load-station` | manipulator | バッチ台車から鋼の棒材ブランクを取り、旋盤の投入ステーションの V ブロックに主軸の高さで置く（0.65 + 0.55 m、2 s） | 肩関節ピークトルク | 180 N·m（estimate） |
| `:coolant-supply-line` | pipe-flow | 水溶性クーラントを機械のタンクから内径 12 mm・6 m の配管（揚程 1.2 m）で工具ノズルへ送る。作業の要求流量を振る | 配管の圧力損失（揚程込み） | 150 kPa（estimate） |

測定の入口: `kbb -M:physics`。全 run が数値を返さなければ exit 2 = **測れなかった**（「異常なし」ではない）。
test: `kbb -M:physai-test`（`test-physai/machinist/physics_spec_test.cljk` が physics.edn の妥当性と全 run の計測を検査する。現時点 24 test / 52 assertion）。

## 測って分かったこと・限界（成長の第一候補）

1. **ブランクの投入**: 肩トルクは 2 kg で 99.5 N·m、10 kg で 166.2 N·m、25 kg で 291.2 N·m（1 kg あたり 8.3 N·m、アーム自重だけで約 83 N·m）。
   限界 180 N·m に達するのは **11.66 kg** —— 鋼の丸棒で φ60 mm × 約 520 mm まで。
2. **クーラント配管**: 圧力損失は 6 L/min（0.1 L/s）で 18.4 kPa、18 L/min で 57.1 kPa、24 L/min で 87.2 kPa、36 L/min で 167.1 kPa（乱流、Re 7,074〜42,441）。
   限界 150 kPa に達するのは **0.562 L/s（約 33.7 L/min）** —— それ以上を要求する作業はこの配管では足りない。
3. **estimate のままの値**: 肩トルク上限 180 N·m（使うアームの仕様書で）、配管に使える揚程 150 kPa（クーラントポンプの性能曲線で置き換える）、
   クーラントの粘度 1.5 mPa·s（希釈率ごとの製品データで置き換える）、ポンプ効率 0.45、配管の粗さ 1.5 µm、アームの寸法・質量。

## 1 反復の手順（成長 tick）

evidence（prompt に注入される）を読み、次の順で **1 つだけ** 選ぶ:

1. evidence が `TESTS-FAIL` / `PROBE-UNMEASURED` → それを直す（最小の差分）。
2. `physics.edn` の `:basis "estimate: ..."` を 1 つ、出典のある値（規格番号・メーカー仕様・法令の条番号と URL）に置き換える。
   出典が取れなければ置き換えない —— 推測で `estimate` を外さない。
3. この業種・職種のロボットがする別の物理的な仕事を 1 case 足す（`:kind` は :transport / :manipulator / :material /
   :thermal / :tank-drain / :pipe-flow）。README の premise と docs から根拠を取る。
4. governor が同じ solver で独立に再計算して、限界を超える action を止める純関数と test を足す（大きい変更。1〜3 が尽きてから）。

作業の仕方（これ以外の経路で main に入れない）:

```
kbb --backend sci ~/github/com-junkawasaki/scripts/physical-ai-bots/tick.cljk branch physai-isco-7223 <slug>   # worktree を切る（path を印字）
# その worktree で編集 → kbb -M:physai-test → kbb -M:physics → git commit
kbb --backend sci ~/github/com-junkawasaki/scripts/physical-ai-bots/tick.cljk land physai-isco-7223 <branch>   # 検証して merge
```

`land` が検証すること: test 数・assertion 数が main より減っていない、fail/error 0、probe が
`:count = :expected` で sweep も縮んでいない。通らなければ merge しない —— そのときは理由を報告して終える。

## 守ること

- **main に直接 push しない。force-push しない。rebase しない。** 着地は `land` だけ。
- **test を弱めて緑にしない**（assert を消す・sweep を減らす・限界を緩めて合格させる）。`land` は数の減少を拒否する。
- **数値を捏造しない。** 物理量は solver が出したものだけ。`:basis` は出典か `estimate:` のどちらかを必ず書く。
- **実機を動かさない。** これはシミュレーションと governor の repo。`:high` / `:safety-critical` な actuation は
  人の承認なしに commit されない設計を崩さない。
- この repo 以外（kotoba-lang/robotics の solver を含む）は編集しない。solver に足りないものは報告に書く。
- 1 反復で終える。報告は: 選んだ候補 / 変えたこと / test 数の前後 / probe の主要量の前後 / land の結果。誇張しない。
