---
name: knowledge-router
description: Route the current Claude conversation content into the appropriate Obsidian Vault(s). Run when the user says "保存", "inbox に入れて", "obsidian に残して", "知識として残して", "疑問として残して", or /knowledge-router. Classifies conversation content into introspective (questions/feelings/inner life → my-knowledge/inbox/) vs external-knowledge (facts/how-tos/definitions → my-research/raw/notes/) segments. Writes to the INBOX/INTAKE layer of each Vault only (never to pages/ directly — that goes through each Vault's own ingest/process skill). Presents a split plan for user approval before writing. Respects both Vaults' CLAUDE.md guardrails. Cross-Vault by design.
---

<!-- version: 1.0 -->

# Knowledge Router

Routes the content of a Claude conversation into the appropriate Obsidian Vault(s). This skill exists because the user's primary interface to their knowledge system is **conversation with Claude**, not direct Obsidian editing. When the user says "save this," this skill decides which Vault gets what, and writes it to the intake layer (not directly to finished pages).

**Scope**: This skill operates **across** Vaults. It is neither owned by `my-knowledge/` nor `my-research/` — it sits above both and routes between them.

---

## Trigger Phrases

Fires on any of:

- 「保存」 / 「残して」 / 「残しておいて」
- 「inbox に入れて」 / 「obsidian に入れて」
- 「obsidian に残して」 / 「obsidian に書いて」
- 「知識として残して」 (→ my-research 寄り)
- 「疑問として残して」 (→ my-knowledge 寄り)
- 「今の会話を保存」 / 「この対話を保存」
- `/knowledge-router`

When the phrase is ambiguous (e.g., just "保存"), proceed with classification. When explicit markers are used (e.g., "知識として"), bias the classification accordingly but still show the plan.

---

## Vault Discovery

**Configured Vaults** (this skill writes to these only):

| Role | Default Path | Purpose |
|---|---|---|
| Introspection Vault | `~/Obsidian/my-knowledge/` | Inner life: questions, feelings, self-discovery |
| Research Vault | `~/Obsidian/my-research/` | External knowledge: facts, techniques, definitions |

**Customization**: If either default path does not exist, ask the user:
> "my-knowledge / my-research Vault のパスを教えてください"

Never write to any other Vault or directory.

---

## Execution Flow

### Phase 1: Read the conversation

Read the current conversation context, focusing on:
- The user's most recent question or statement (the seed)
- Claude's response (the content)
- Any multi-turn exchange that constitutes one logical unit

Do NOT include earlier unrelated conversation turns unless the user explicitly says "全ての会話を保存".

### Phase 2: Segment & classify

Break the conversation into one or more semantic segments, then classify each:

**Introspection signals → my-knowledge:**
- First-person feelings ("不安", "気になる", "わからない気持ち")
- Questions about self or values ("どう生きるべきか", "なぜ私は")
- Journal-like content ("今日は〜", "さっき散歩中に")
- Meta-questions about the user's own thinking

**External-knowledge signals → my-research:**
- Third-person factual explanation ("〜とは〜である")
- Technical mechanisms, step-by-step how-tos
- Named entities, concepts, tools, papers
- Comparison of external things

**Split signals → both Vaults:**
- A single exchange containing both a personal question and a factual explanation
- Example: "RAG って何？ ... そうか、でも自分で作れる気がしない" = 知識 part + 内省 part

### Phase 3: Naming & format

Generate filenames following each Vault's conventions.

**my-knowledge/inbox/ filenames**:

| Prefix | Used when | Format |
|---|---|---|
| `q-` | New question / inquiry | `q-{短タイトル}-YYYY-MM-DD.md` |
| `s-` | Source material description | `s-{ソース名}-YYYY-MM-DD.md` (rare in this skill) |
| `j-` | Journal-like / mood / unfiltered | `j-{短タイトル}-YYYY-MM-DD.md` |

**my-research/raw/notes/ filenames**:

```
claude-{トピック}-YYYY-MM-DD.md
```

If a file with the same name already exists, suffix with `-2`, `-3`, etc.

### Phase 4: Draft content

#### For my-knowledge/inbox/ (質問・内省・ジャーナル)

Format — preserve the user's own voice:

```markdown
---
source: claude-conversation
created: YYYY-MM-DD HH:MM
tags: [router-imported]
---

# {問いまたは短タイトル}

> **なぜ気になったか**: （推測: ... / 本人記載なしの場合は空欄のままにして /process-inbox で埋める）

## 発端

{ユーザの発言そのまま引用、or 要約}

## Claude との対話で出てきたこと

{自分の気持ちに関わる部分のみ抜粋}

## 残る問い

- (もしあれば)
```

**重要**: 事実説明部分は転記しない。内省に関わる部分だけを残す。

#### For my-research/raw/notes/ (外部知識)

Format — preserve factual content for later wiki-ingest:

```markdown
---
source: claude-conversation
source_type: conversation
captured: YYYY-MM-DD HH:MM
topic: "{トピック}"
tags: [router-imported]
---

# {トピック}（Claude対話より）

## 要約

{1-2 文で何について話したか}

## 概要説明

{Claude が話した事実ベースの説明。出典や参照先があれば明示}

## 関連概念

- {概念1}
- {概念2}

## wiki-ingest 向けメモ

- 本メモは Claude との対話から抽出した二次ソース
- 可能なら一次ソース（論文・公式ドキュメント等）で裏取りしてから pages/ へ昇華することを推奨
```

**重要**: Claude の発言内容は完全にそのまま貼り付けず、**事実部分を整形**する。冗長な補足・絵文字・対話調は削る。

### Phase 5: Present the plan

Before writing, show the plan:

```
以下のように分けて保存します:

[my-knowledge/inbox/q-XXX-2026-04-18.md]
  → 「XXX」という問い（内省含む）
  → プレビュー: {先頭50字}...

[my-research/raw/notes/claude-YYY-2026-04-18.md]
  → 「YYY」の事実説明
  → プレビュー: {先頭50字}...

この内容で保存しますか？
  A) はい、保存
  B) my-knowledge のみ
  C) my-research のみ
  D) ファイル名を変更したい
  E) 中身を調整したい（どう変える？）
  F) やめる
```

Wait for user confirmation. Do NOT write without approval.

### Phase 6: Write the files

Once approved:

1. Write each file to its target path
2. Append to each affected Vault's `log.md`:
   - `my-knowledge/log.md`:
     ```
     {YYYY-MM-DD HH:MM} router | {filename}
     ```
   - `my-research/log.md`:
     ```
     ## [YYYY-MM-DD] router | raw/notes/{filename}
     ```
3. Atomic: if either write fails, report and leave both in a known state

### Phase 7: Report & suggest next steps

After writing, suggest:

```
保存完了:
  ✓ my-knowledge/inbox/q-XXX-YYYY-MM-DD.md
  ✓ my-research/raw/notes/claude-YYY-YYYY-MM-DD.md

次のおすすめ操作:
  - my-knowledge で /process-inbox を走らせて pages/ に整理
  - my-research で wiki-ingest して pages/concepts や entities に整理

今すぐ実行しますか？（各 Vault の CLAUDE.md に基づき判断）
```

Do NOT auto-invoke /process-inbox or wiki-ingest. User must confirm.

---

## Guardrails

1. **NEVER** write directly to `pages/` of either Vault. Always write to the inbox/intake layer only. Page-level content must flow through each Vault's own ingest skill.
2. **NEVER** write to Vaults other than `my-knowledge/` and `my-research/`. Ignore `data-wiki/` and any other Obsidian Vaults.
3. **NEVER** write without showing the plan and getting approval.
4. **RESPECT** each Vault's CLAUDE.md. Read them to check for additional guardrails (e.g., no personal data, no mass changes).
5. **PRESERVE** attribution: every file written by this skill has `source: claude-conversation` in frontmatter and `tags: [router-imported]`.
6. **DO NOT** copy entire Claude responses verbatim. Extract and restructure:
   - my-knowledge: preserve user's own voice, trim Claude's explanatory scaffolding
   - my-research: preserve factual content, restructure into knowledge-note format
7. **STOP** and ask when classification is unclear. Better to pause than misroute.
8. **DO NOT** auto-invoke follow-up skills (/process-inbox, wiki-ingest, etc.). Suggest them but wait for user action.

---

## Classification Edge Cases

### Case: "プロジェクト決定" や設計議論

Example: "my-research の設計はこうしよう"

This is meta-conversation about the user's own projects.

- **Default routing**: my-knowledge/inbox/ with `j-` prefix (journal-like self-reflection about work)
- **Alternative**: my-research/raw/notes/ if the content is primarily technical specification
- **Ask user** if clearly mixed

### Case: ユーザの「気分」表明のみ

Example: "今日疲れた" / "ちょっと落ち込んでる"

- **Route**: my-knowledge/inbox/ with `j-` prefix
- **Do NOT** try to convert this into a question
- **Do NOT** write extensively — keep it short and preserved as-is

### Case: 事実確認クエリ

Example: "日本の人口は？" → "1.2億人です"

- **Typically do NOT save** unless user explicitly requests
- Such trivial Q&A has low archival value
- If user insists, route to my-research/raw/notes/ but keep very short

### Case: 完全に曖昧な保存指示

Example: User says only "保存" after a long multi-topic conversation

- Ask: "どの部分を保存しますか？ 会話全体？ 最後のトピック？"
- Do NOT guess and write wholesale

### Case: 複数Vaultのどちらでもない内容

Example: Business decision, meeting notes, work planning

- These do not belong to either my-knowledge (too external) nor my-research (too personal/work)
- **Ask user**: "これは business/work 系なので、my-knowledge か my-research のどちらに入れますか？ あるいは別のVaultに置きたいですか？"
- **Default fallback** if user chooses: my-knowledge/inbox/j- (as a journal entry)

---

## Usage Examples

### Example 1: Mixed conversation

```
Context:
  [User] RAG って何？
  [Claude] RAG は... (800字の事実説明)
  [User] なるほど。でも自分で作れる気がしないな
  [Claude] ...（励まし系応答）
  [User] 今の会話を保存して

Skill flow:
  Phase 1: Read above 5 turns
  Phase 2: Classify into 2 segments:
    - Segment A (3 turns): RAG factual explanation → my-research
    - Segment B (2 turns): "作れる気がしない" inner voice → my-knowledge
  Phase 3: Generate names
    - my-research/raw/notes/claude-RAG基本-2026-04-18.md
    - my-knowledge/inbox/q-RAGを自分で作れる気がしない-2026-04-18.md
  Phase 4: Draft both files
  Phase 5: Present plan:
    "以下のように分けます:
       [my-research] claude-RAG基本-2026-04-18.md (事実説明)
       [my-knowledge] q-RAGを自分で作れる気がしない-2026-04-18.md (内省)
     これで保存してよいですか？"
  Phase 6: User says "はい"
    Write both files
    Append to both log.md
  Phase 7: Report:
    "保存完了。
     my-research は wiki-ingest で pages/ に整理できます。
     my-knowledge は /process-inbox で pages/ に育てられます。"
```

### Example 2: Pure introspection

```
Context:
  [User] 最近引退のことを考えると不安になる
  [Claude] 不安の中身は？ (dialogue)
  [User] ... (more sharing)
  [User] 今の対話を保存

Skill flow:
  Phase 2: All content is introspective → my-knowledge only
  Phase 3: my-knowledge/inbox/q-引退への不安-2026-04-18.md
  Phase 4: Draft preserving user's voice
  Phase 5: "my-knowledge のみへの保存になります:
            q-引退への不安-2026-04-18.md
            これでよいですか？"
```

### Example 3: Pure factual

```
Context:
  [User] Kubernetes の Pod と Container の違いは？
  [Claude] (technical explanation)
  [User] 残して

Skill flow:
  Phase 2: All factual → my-research only
  Phase 3: my-research/raw/notes/claude-Kubernetes-Pod-vs-Container-2026-04-18.md
  Phase 4: Restructure into knowledge-note format
  Phase 5: Present plan
  Phase 6: Write
  Phase 7: Suggest wiki-ingest
```

### Example 4: Ambiguous save request

```
Context:
  30 turns of conversation across multiple topics
  [User] 保存して

Skill flow:
  Phase 1: Detect ambiguity
  Phase 2: Instead of classifying all 30 turns, ask:
    "会話が長いので、どこを保存しますか？
       A) 最後の1トピック（Kubernetes について）
       B) 2つのトピック分（引退の不安 + Kubernetes）
       C) 全部
       D) この part を保存したい（指定）"
  After user clarifies, proceed with Phase 2 properly
```

### Example 5: User specifies Vault explicitly

```
Context:
  [User] RAG について教えて
  [Claude] (explanation)
  [User] これ知識として残して

Skill flow:
  "知識として" marker → skip classification, route to my-research only
  Phase 3-7: proceed
```

---

## Interaction with Other Skills

- **Before**: This skill is typically the **first** write operation in a conversation. It does not consume output from another skill.
- **After**: This skill's output (inbox/, raw/notes/) becomes **input** to:
  - `/process-inbox` in my-knowledge → pages/ and journal/ in my-knowledge
  - `wiki-ingest` in my-research → pages/ in my-research
- **Do not directly invoke** these follow-up skills. Only suggest them. The user decides when to run them.

---

## Post-conditions

After a successful run:

1. New files exist in inbox/ (my-knowledge) and/or raw/notes/ (my-research)
2. Each affected Vault's log.md has a `router | ...` entry
3. No other files are modified
4. User knows what was saved where and what to do next

Cleanup on failure:
- If write to first Vault succeeded but second failed, notify user and leave first file in place
- Do NOT attempt rollback (the partial save might still be useful)
