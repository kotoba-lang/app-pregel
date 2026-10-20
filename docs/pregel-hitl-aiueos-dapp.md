# app-pregel（HITL dispatch）— itonami（dapp）上の分散型設計

状態: 第 1 スライス実装（2026-10-20）。正本はこの文書。
追跡: Linear JUN-128（親 JUN-116）。

## 位置づけ

app-pregel は Pregel スタイルのグラフ計算面（outlook_triage / pregel_triage /
projector_* の LangGraph twin 群 + HITL store seam）。実装の大半は外部
kotodama パッケージにあり、この repo は **dispatch 面と seam** を持つ。

## このスライス: HITL 判定コア

| 層 | 実体 | 状態 |
|---|---|---|
| 判定コア guest | `aiueos/pregel_hitl.cljk` — hitl-admission（**verdict は pending の thread にだけ**）/ store-safe（**既定は read-only / empty = offline で安全**）/ swap-judgment（**store 差替のみ、ルーティング不変**）/ ledger-ok（**正本の状態は append-only 監査台帳**） | **実装済み・実測** |
| 記述子 | `aiueos/pregel-hitl.edn` | **実装済み** |
| parity | `lg-clj/test/pregel/hitl_parity_test.cljk` — 正本（store.cljk の HITL seam）の規約一致検査 | **実装済み・実測** |

実測: `amu test aiueos/pregel_hitl.cljk` → **15/15**（jvm-kir / js / wasm）。
parity → 8 assertions / 0 failures。

## 正本の正確な規則（guest はこれに追従 —— store.cljk の明文）

- HITL dispatch: `list-pending-gray / list-pending-drafts → get-*-item →
  apply-verdict / apply-draft-verdict`。**pending の thread にだけ verdict**。
- **default-store は「Read-only / empty default（no RisingWave, no live
  kotodama）。Safe offline。」** —— 既定は生の依存に触らせない。
- **actor swap パターン**: 本番は `*store*` を kotoba-Datom-log アダプタに
  差し替えるが「**WITHOUT touching the routing**」。
- **正本の状態は append-only audit ledger**（kotoba Datom log）。RisingWave は
  substrate 境界で**禁止**（ADR-2605262130 で非推奨）—— この port はその
  偏差を seam で吸収する。

## 未実装境界

- verdict 値の閉じた集合（外部 kotodama パッケージ側にあり未特定）。
- projector_lifecycle / projector_ops の判定（projector の状態遷移）。
- LangGraph twin の graph 進行判定（injection seam の中身は host）。
- mon（DNA）化、aiueos session 載せ（fleet 共通）。
