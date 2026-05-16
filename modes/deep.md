# Mode: deep -- Deep Research Prompt

## Language

This mode is **user-facing**: the output is a research/interview-prep doc the
user reads, not application content sent to the company. Override the shared
"language of the JD (EN default)" rule for this mode and resolve the output
language in this order:

1. **User prompt language** -- if the user wrote `/career-ops deep` or the
   surrounding chat in Spanish, French, German, Japanese, etc., emit the doc in
   that language.
2. **`config/profile.yml`** -- if `language.modes_dir` is set (`modes/de`,
   `modes/fr`, `modes/ja`, etc.), prefer that locale.
3. **JD language** -- only as a last resort, when the user prompt has no
   language signal, such as a bare URL with no surrounding chat.

The template below is written in English as the default scaffold. Translate it
to the resolved output language before presenting when the user or profile asks
for another language.

Generate a structured prompt for Perplexity/Claude/ChatGPT with 6 axes:

```markdown
## Deep Research: [Company] -- [Role]

Context: I am evaluating an application for [role] at [company]. I need
actionable information for interview preparation.

### 1. AI Strategy
- Which products or features use AI/ML?
- What is their AI stack? Include models, infrastructure, and tools.
- Do they have an engineering blog? What do they publish?
- Have they published papers or given talks about AI?

### 2. Recent Moves (last 6 months)
- Relevant AI/ML/product hires?
- Acquisitions or partnerships?
- Product launches or pivots?
- Funding rounds or leadership changes?

### 3. Engineering Culture
- How do they ship? Include deploy cadence and CI/CD.
- Monorepo or multirepo?
- Which languages and frameworks do they use?
- Remote-first or office-first?
- What do Glassdoor/Blind reviews say about engineering culture?

### 4. Likely Challenges
- What scaling problems do they have?
- Reliability, cost, or latency challenges?
- Are they migrating anything? Infrastructure, models, or platforms?
- What pain points do people mention in reviews?

### 5. Competitors and Differentiation
- Who are their main competitors?
- What is their moat or differentiator?
- How do they position themselves against competitors?

### 6. Candidate Angle
Given my profile (read from cv.md and profile.yml for specific experience):
- What unique value do I bring to this team?
- Which of my projects are most relevant?
- What story should I tell in the interview?
```

Personalize every section with the specific context of the evaluated role, in
the language resolved by the **Language** section above.
