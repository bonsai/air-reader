# air-reader

空気を読むエージェント — 状況を観察し、文脈と関係性から意図を仮説化し、次の一手を提案する。

> **Status:** Design / MVP planning  
> **Principle:** 空気を読んだと断定しない。観察された事実・解釈・不確実性を分離する。

## 目的

「空気を読む」を、根拠のない感情当てではなく、状況に応じた行動選択として扱う。

エージェントは、会話や作業の文脈から次を支援する。

- いま何が起きているかを観察・要約する
- 明示された発言と、推測された意図を区別する
- 人・役割・期待・制約・関係性をモデル化する
- 話す、聞く、待つ、質問する、提案する、介入しない、の候補を比較する
- 不確実性が高い場合は確認を優先する
- 結果や本人からのフィードバックを使い、判断を評価する

## 設計原則

1. **Evidence before inference** — 観察事実と解釈を混ぜない。
2. **Hypothesis, not truth** — 意図や感情は仮説として複数候補を持つ。
3. **Ask before assuming** — 判断によって相手に影響する場合、必要に応じて確認する。
4. **Silence is an action** — 待つ・介入しないことも有効な選択肢とする。
5. **Context is scoped** — 状況・参加者・時間範囲を明示し、文脈を無制限に持ち越さない。
6. **Human agency** — 相手を操作するのではなく、理解と協調を助ける。
7. **Privacy by default** — 必要最小限の情報を処理し、秘密の心理プロファイルを作らない。
8. **Learn from feedback** — 正解を勝手に定義せず、本人の訂正と観測可能な結果を使う。

## Ontology

### World layer

| Entity | Meaning |
|---|---|
| Situation | 場面、目的、時間、環境、制約 |
| Participant | 会話・活動に参加する主体と、その場で明示された役割 |
| Utterance | 発言、質問、回答、依頼、拒否 |
| Signal | 沈黙、発話順、割り込み、応答時間などの観測可能な信号 |
| Norm | 明示されたルール、共有された期待、場面固有の慣習 |
| Relationship | 役割・依存関係・合意など、根拠がある関係 |
| Observation | 時刻・出典・対象を持つ観察記録 |
| Hypothesis | 意図・ニーズ・場の状態についての暫定解釈 |
| ActionCandidate | 次に取り得る行動 |
| Outcome | 行動後に観察された結果 |
| Feedback | 本人による訂正・評価・希望 |

### Action layer

各アクションは対象、目的、根拠、前提条件、リスク、権限、期待する効果、検証方法を持つ。デフォルトは**提案のみ**。発言送信・メッセージ送信・外部システム変更などは、明示的な許可なしに実行しない。

## Agent architecture

```text
Input / Context
      |
      v
Observation Extractor
      |
      v
Evidence Ledger -----> Privacy / Scope Filter
      |
      v
Situation & Relationship Model
      |
      v
Hypothesis Generator
      |
      v
Action Candidate Generator
      |
      v
Decision Matrix
      |
      v
Confidence / Risk / Authorization Gate
      |
      +---- uncertain ----> Ask / Defer / No-op
      |
      v
Recommendation (default; no autonomous sending)
      |
      v
Outcome + Human Feedback
      |
      v
EVOPS Evaluation / Regression Set
```

### Agent responsibilities

- **Observer**: 入力を観察単位に分解し、根拠と時刻を記録する。
- **Context Builder**: 場面・役割・明示された規範・直近の履歴を整理する。
- **Interpreter**: 競合する解釈を複数生成し、支持・反証の根拠を示す。
- **Decision Agent**: 行動候補を、期待効果・誤解リスク・可逆性・コストで比較する。
- **Safety Gate**: プライバシー、権限、影響範囲を確認する。
- **Evaluator (EVOPS)**: 根拠の追跡性、過剰な断定、確認行動、結果を評価する。

MVPでは、これらを複数の独立したLLMエージェントにする必要はない。まず一つの処理パイプラインとして実装し、役割は契約で分離する。

## Decision model

候補行動は以下の観点で比較する。

- Goal fit: 明示された目的に合うか
- Evidence support: 根拠があるか
- Misinterpretation risk: 誤解した場合の損害
- Reversibility: 取り消しや修正が容易か
- Intrusiveness: 相手への介入が過剰でないか
- Information value: 確認することで不確実性を減らせるか

数値スコアだけで自動決定しない。重大な影響がある場合や根拠が弱い場合は、質問・保留・人間への確認を優先する。

## Event schema (JSONL concept)

1行を1イベントとし、追記型で記録する。実際のスキーマはJSON Schemaとして別途固定する。

```json
{
  "schema_version": "0.1.0",
  "event_id": "evt_example_001",
  "session_id": "session_example",
  "timestamp": "2026-01-01T00:00:00Z",
  "event_type": "hypothesis.created",
  "situation_id": "situation_example",
  "subject_ref": "participant_a",
  "observation_refs": ["obs_001"],
  "claim": "参加者は追加説明を求めている可能性がある",
  "claim_type": "hypothesis",
  "supporting_evidence": ["obs_001"],
  "counter_evidence": [],
  "confidence": 0.45,
  "alternatives": [
    "考えを整理している",
    "発言する機会を待っている"
  ],
  "recommended_next_step": "ask_neutral_question",
  "requires_human_approval": true,
  "privacy_class": "restricted"
}
```

これは説明用の例であり、確定したAPI契約ではない。confidenceはモデルの自己申告にすぎないため、校正済み確率と同一視しない。個人情報・生の会話内容を保存する必要性は別途評価する。

## MVP scope

### In scope

- テキスト会話・明示的な場面説明を入力する
- 観察事実と解釈を分離して表示する
- 競合する仮説を2つ以上提示できる
- 根拠、不確実性、反証材料を示す
- 次の行動候補を複数提示する
- 確認・保留・介入しない選択肢を含める
- 人間のフィードバックを記録し、評価用ケースにできる
- JSONLイベントと再現可能なテストケースを出力する

### Out of scope for MVP

- 表情や声から内面・性格・意図を断定すること
- 無断での録音・監視・個人の心理プロファイリング
- 相手を欺く、操作する、同調圧力をかけるための利用
- 人間の承認なしに対人メッセージを送信すること
- 単一の「空気スコア」で人や集団を順位付けすること

## Evaluation (EVOPS)

ゴールは「正解の感情を当てる」ことではなく、根拠に基づいて安全で適切な次の行動を選べること。

- Evidence traceability: 主張に根拠が紐づく割合
- Fact / inference separation: 事実と推測の混同率
- Calibration: 信頼度と実際の正確さの対応
- Alternative coverage: 妥当な別解釈を提示できる割合
- Clarification quality: 必要な場面で中立的な確認を選ぶ割合
- Overconfident assertion rate: 根拠不足なのに断定する割合
- Intrusive action rate: 不必要な介入を提案する割合
- Human correction rate: 人間の訂正頻度と訂正の種類
- Privacy violations: 秘密情報の不必要な保存・出力（許容目標はゼロ）

テストセットには、曖昧な沈黙、皮肉、文化差、役割の非対称性、意見対立、情報不足、複数の合理的解釈があるケースを含める。本人のフィードバックを唯一の普遍的な正解とせず、ケースの目的と評価基準を明記する。

## Implementation sequence

1. Ontology・用語・境界を確定する
2. Observation / Hypothesis / ActionCandidate のJSON Schemaを作る
3. JSONLサンプルと曖昧な状況の評価ケースを作る
4. テキスト入力から根拠付きの仮説・行動候補を出す最小パイプラインを実装する
5. EVOPSで断定率・根拠追跡性・確認行動を評価する
6. フィードバックによる回帰テストを追加する
7. 必要性と同意が確認できた範囲だけ、音声・映像・外部ツールを検討する

## Repository contract

Ontology → Schema → API Contract → Implementation → Agent → Workflow → View

- GitHubを設計・仕様・評価ケースの正本にする。
- 言語やモデルを変えてもスキーマ/API契約は変えない。
- 観察事実、仮説、判断、実行、結果を別イベントとして扱う。
- データがないことを「問題がない」と解釈しない。
- 実装の有無と、将来の構想をREADMEで区別する。

## Open questions

- 最初の対象は、1対1の会話、複数人の会議、接客、チーム作業のどれか
- 入力はテキストのみから始めるか
- どのフィードバックを評価上の根拠とするか
- セッションデータの保存期間と削除方針
- ローカル処理とクラウド処理の境界

---

**最初の実装目標:** テキストで状況を渡すと、エージェントが「観察された事実 / 複数の解釈 / 根拠と不確実性 / 推奨する次の一手 / 確認すべきこと」を返す。自動実行より先に、判断の質を評価可能にする。
