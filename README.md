# Personal Headhunter

A private job-search pipeline that finds, screens, and tracks openings across a curated list of employers, then hands a short, evidence-backed shortlist to Claude for review every morning.

> **Status:** Phase 1 in progress. Nothing below runs yet unless it is marked as working in the Build status table.

## Why this exists

General job boards are noisy, often out of date, and full of reposted roles. This system goes straight to each employer's own careers feed, filters out roles that can't work for me before anyone reads them, and keeps a record of every role it has seen, so nothing gets reviewed twice and nothing is reported as closed by mistake.

## How it works

```
GitHub Actions (about 6:30 a.m. ET)          Claude (about 7:50 a.m. ET)
┌─────────────────────────────────┐          ┌──────────────────────────────┐
│ 1. Collect: employer ATS feeds  │          │ 5. Read the full job text    │
│    plus Indeed keyword search   │          │ 6. Assign a visa queue       │
│ 2. Normalize and dedupe         │  ──────► │ 7. Score fit with evidence   │
│ 3. Diff against yesterday       │  repo    │ 8. Check every link          │
│ 4. Apply hard filters           │  data    │ 9. Update tracker and digest │
└─────────────────────────────────┘          └──────────────────────────────┘
```

### 1. Collection

| Source | Method | Notes |
|---|---|---|
| Greenhouse, Lever, Ashby | Official public job-board APIs | Stable and documented |
| SmartRecruiters, Workable, Recruitee | Public posting APIs | Stable |
| Workday | Careers-site JSON endpoint (`/wday/cxs/...`) | Undocumented, so it can change without notice |
| Indeed | [JobSpy](https://github.com/speedyapply/JobSpy) keyword search | Discovery only; every role is confirmed on the employer's own site |
| iCIMS, Taleo, Oracle, custom sites | Manual browser rotation | No public feed |

Every run writes a **coverage report** that labels each employer as fully extracted, partial, unsupported, or failed. A gap in coverage is shown as a gap. It is never reported as "no openings."

### 2. Change tracking

- One record per role, keyed by employer and requisition ID.
- Each role is marked **new**, **changed**, or **missing**.
- A role counts as **closed** only after two missed runs in a row while its source is otherwise healthy.

### 3. Sponsorship data (refreshed quarterly)

- Public DOL LCA disclosure data, filtered to relevant occupation codes (engineering, software, analyst roles).
- Used to score each employer's filing history for similar titles and to find new employers worth adding.
- Filing history is a **research lead**, not proof that a specific role will sponsor.

### 4. Hard filters (code, before review)

These roles are rejected automatically:

- Non-US locations
- Senior roles that clearly require 5 or more years of experience
- Explicit restrictions: no current or future sponsorship, no OPT/CPT/STEM OPT, citizenship or clearance required

For every rejection, the exact sentence that triggered it is saved, along with the source and the date checked.

### 5. Review queues

| Queue | Meaning |
|---|---|
| Fully verified | Evidence supports all work-authorization requirements |
| Conditional | The posting explicitly supports sponsorship; STEM OPT support is unconfirmed |
| Verification lead | Sponsorship is unknown; kept off the application shortlist |
| Rejected | Explicit incompatibility, or a clear qualification or timing mismatch |

Silence about sponsorship is treated as **unknown**. It never counts as approval.

## Repository layout (planned)

```
config/
  employers.yaml        # employer list: ID, name, track, priority, hold status
  sources.yaml          # resolved ATS type and endpoint for each employer
  filters.yaml          # titles, locations, seniority, restriction phrases
collector/
  adapters/             # one module per ATS
  run.py                # collect, normalize, diff, filter
sponsorship/
  lca_refresh.py        # quarterly DOL data refresh and employer scoring
data/
  jobs.json             # role ledger (state across runs)
  coverage/             # daily coverage reports
  candidates/           # daily filtered roles for review
  rejected.json         # rejections with evidence
.github/workflows/
  daily.yml             # scheduled collection
  lca.yml               # quarterly sponsorship refresh
```

## Build status

| Phase | Scope | Status |
|---|---|---|
| 1 | Collectors, employer seed list, coverage report | In progress |
| 2 | Sponsorship data, hard filters, daily diff | Not started |
| 3 | Morning review, tracker, digest | Not started |
| 4 | Weekly browser pass for unsupported employers | Not started |

A phase is marked done only after it has been tested against the real employer list. Phase 3 also requires three clean mornings in a row and 10 randomly sampled links that open correctly.

## Ground rules

- **No automatic applications, messages, or account creation.** The system finds and tracks. I decide and act.
- **Report what was actually checked.** A reachable homepage is not a scraped job board, and a status of 200 does not mean the application page works.
- **Excluded employers stay excluded** until I say otherwise.
- **Personal data stays out of this repo.** That includes resumes, contact exports, and immigration details.

## Known limitations

- GitHub's scheduled runs can be delayed or skipped under load. The morning review checks how fresh the data is and flags it when it's stale.
- The Workday endpoint is undocumented, and individual employers occasionally go down for maintenance.
- Most job boards return about 1,000 results per search at most.
- LinkedIn scraping is disabled by default because LinkedIn blocks automated access.

## Acknowledgments

The design draws on patterns from [JobSpy](https://github.com/speedyapply/JobSpy), [career-ops](https://github.com/santifer/career-ops), and published write-ups on public ATS feeds and on running zero-cost daily job pipelines with GitHub Actions.
