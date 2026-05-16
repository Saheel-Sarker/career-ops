# Mode: tracker -- Application Tracker

Read and show `data/applications.md`.

**Tracker format:**

```markdown
| # | Date | Company | Role | Score | Status | PDF | Report |
```

Canonical states: `Evaluated` -> `Applied` -> `Responded` -> `Interview` ->
`Offer` / `Rejected` / `Discarded` / `SKIP`

- `Applied` = the candidate submitted the application
- `Responded` = a recruiter/company contacted the candidate and the candidate replied
- `Interview` = the candidate is in an interview process
- `Offer` = the company extended an offer
- `Rejected` = the company rejected the candidate
- `Discarded` = the candidate discarded the role or the posting closed
- `SKIP` = poor fit; do not apply

If the user asks to update a status, edit the corresponding row.

Also show statistics:
- Total applications
- Count by status
- Average score
- Percentage with PDF generated
- Percentage with report generated
