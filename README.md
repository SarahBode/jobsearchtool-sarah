# Sarah's Job Search Tool

A persistent, re-runnable pipeline that searches company ATS boards, screens
each listing against Sarah's Blueprint (including a scam/impersonation check),
and produces a tailored **resume + cover letter + prefilled answers** for every
Strong/Partial/Worth-a-Look match, plus a roster (CSV, iPad-friendly) you can
act on manually.

> **This tool never submits anything.** Every application is a manual, human
> action by you. There is no autosubmit path anywhere in the code.

> **On an iPad?** You don't install anything. See **[IPAD_SETUP.md](IPAD_SETUP.md)**:
> open the repo in Claude Code on the web, type `/find-jobs`, and the roster and
> tailored files come straight to you in the chat.

---

## What each run does

```
load state  ->  fetch new listings from prioritized ATS boards
            ->  scam/impersonation check (runs first)
            ->  screen line-by-line against the Blueprint
            ->  annualize hourly pay before the $50k floor comparison
            ->  near-misses that fit everything else get flagged "Worth a Look"
            ->  for Strong/Partial/Worth-a-Look (or explicit overrides): tailor
                resume + cover letter + prefilled answers
            ->  append output/roster.csv
            ->  update state.json (idempotent: only NEW jobs each run)
            ->  print a run summary
```

## Install

```bash
pip install -r requirements.txt
```

## Run

**Simplest (iPad or anywhere):** in a Claude Code session, type **`/find-jobs`**
(or "run the job search"). The `find-jobs` skill runs the engine, adds live
Glassdoor/scam judgment, delivers the roster and files to you in chat, and
commits state back to the repo. See [IPAD_SETUP.md](IPAD_SETUP.md).

The two lower-level ways to run, by design:

**1. Deterministic engine (no network judgment):**
```bash
python run_pipeline.py
```
Fetches, screens, annualizes pay, tailors, and writes the roster. Glassdoor/news
consensus is left as a clearly-marked `[web check pending]` note because raw
Python cannot browse the web.

**2. Claude-driven run (recommended, higher quality):**
```bash
claude -p "run job search pipeline"
```
Claude runs the same engine **and** performs the web-dependent screening steps:
verifies each company is real and its operations match the JD, pulls a
multi-source Glassdoor + recent-news consensus (weighing for real signal, noting
where reviews skew alarmist or under-sell), and can escalate a listing to
`POSSIBLE SCAM` on judgment the heuristics miss. See `PIPELINE.md`.

**Screen a single pasted JD** (quick check, no tailoring):
```bash
python run_pipeline.py --screen-file path/to/jd.txt
```

**Override a Not-a-Match** (tailor anyway; verdict is NOT changed):
```bash
python run_pipeline.py --override "greenhouse:slug:12345"
```

**Recommend a near-miss** (flexible judgment; adds a `Worth a Look (recommended)`
row with a tailored packet and an honest one-line rationale):
```bash
python run_pipeline.py --recommend "greenhouse:slug:12345" --why "Pay is $2k under the floor, but it is a special-needs services coordinator role with growth room."
```
The deterministic engine also auto-surfaces near-misses (pay within 10% of the
floor when mission, level, and remote all fit) as `Worth a Look`. Tuning lives
under `flex_recommendations` in `config/blueprint.yaml`. Scam-flagged listings
can never be recommended.

## Configure

| File | What it controls |
|---|---|
| `config/blueprint.yaml` | The screening filter: must-haves, absolute cons, salary floor, level, mission themes, and `flex_recommendations` (the Worth-a-Look near-miss rules). **Edit this to change how jobs are judged.** |
| `config/keywords.yaml` | Role title keywords, mission keywords, and the **seed boards** to poll. Add `{greenhouse: [slug], lever: [slug], ashby: [slug], smartrecruiters: [slug], recruitee: [slug]}` slugs to find more roles. |
| `data/base_resume.json` | Your real resume. The honest source material. |
| `data/honest_gaps.json` | Claims that must never be implied. Enforced automatically. |
| `data/verified_strengths.json` | Honest raw material mapped to real resume bullets. |

### Adding job sources

The pipeline is **keyword-driven** but needs concrete public boards to read.
A starter set of established mission-driven orgs ships in the config (ACLU,
Wikimedia, Khan Academy, Code.org, Candid, DonorsChoose, Newsela, GiveWell, CZI
on Greenhouse; **KIPP Public Schools** on SmartRecruiters; **MSF / Doctors
Without Borders** on Recruitee); add or prune slugs under `seed_boards` in
`config/keywords.yaml`:

```yaml
seed_boards:
  greenhouse:      ["examplenonprofit"]  # boards-api.greenhouse.io/v1/boards/<slug>/jobs
  lever:           ["exampleorg"]        # api.lever.co/v0/postings/<slug>
  ashby:           ["ExampleOrg"]        # api.ashbyhq.com/posting-api/job-board/<slug>
  smartrecruiters: ["Kipp"]              # api.smartrecruiters.com/v1/companies/<slug>/postings
  recruitee:       ["msf"]               # <slug>.recruitee.com/api/offers/
  fourdayweek:     ["coordinator"]       # 4dayweek.io/api/jobs?q=<query> (search terms, not slugs)
```

Five public **ATS** providers are supported (each a documented, public job-board
feed that powers a company's own careers page, no login and no API key) plus one
public **job board**:

- Greenhouse: `boards.greenhouse.io/<slug>` or `job-boards.greenhouse.io/<slug>`
- Lever: `jobs.lever.co/<slug>`
- Ashby: `jobs.ashbyhq.com/<slug>`
- SmartRecruiters: `jobs.smartrecruiters.com/<slug>/...`
- Recruitee: `<slug>.recruitee.com` (the jobs-board subdomain)
- **4dayweek.io** (job board): entries under `fourdayweek` are **search
  queries**, not company slugs. The fetcher queries `4dayweek.io/api/jobs`
  filtered to US locations and title-filters the results like every other
  source. 4dayweek locks the full JD text, salary, and external apply link
  behind each job page, so those roster columns stay light for 4dayweek rows and
  you open the linked listing for the rest; the salary is screened as
  "unknown", never fabricated.

For the ATS providers, find a company's slug from its careers page URL (shown
above). LinkedIn, Indeed, ZipRecruiter, and Glassdoor job feeds are
**manual-search-required** (login walls / anti-scraping ToS) and are reported as
such every run rather than scraped.

## Outputs

```
output/
  resumes/       resume_<Company>_<Role>.docx        (one page, tailored)
  coverletters/  coverletter_<Company>_<Role>.docx   (auto-generated per resume)
  prefilled/     prefilled_<Company>_<Role>.md        (common ATS answers)
  roster.csv     the actionable table (opens in Numbers/Excel on iPad)
```

The roster is a CSV with one row per listing: Date Found, Company, Role,
Verdict, **Why** (plain-language reason for the verdict), Salary, Apply Link,
Glassdoor Consensus, Flagged Gaps, the three file names, Scam Flag, Override.

On the web/iPad workflow the repo is the tool's memory, so `state.json`,
`roster.csv`, and the generated files are committed each run (they persist across
ephemeral containers and are a download fallback). You receive them directly
in chat, so you never have to browse GitHub. Prefer to keep generated files out
of git? Re-add the `output/` globs to `.gitignore`.

## Guardrails baked in

- **No autonomous submission.** Ever.
- **No fabrication.** Tailoring only reorders/selects your real bullets. A
  test asserts every tailored bullet exists verbatim in the base resume.
- **Honest-gaps enforcement.** `data/honest_gaps.json` phrases are blocked from
  resumes outright; if a JD requires a gap area, the cover letter names it in
  1-2 constructive sentences instead of writing around it.
- **No em dashes** in resumes/cover letters (en dashes in date ranges are fine).
- **One page** resumes: tailoring never adds content, so length equals the base.
- **Idempotent:** `state.json` tracks seen job IDs; re-runs only report new ones.
- **Respects ToS:** only public, documented ATS job-board APIs are read, with a
  descriptive User-Agent and polite rate limiting. No CAPTCHA solving, no login.

## Tests

```bash
python tests/test_smoke.py     # or: pytest tests/
```
Covers salary annualization, verdict logic, scam detection (and no false
positive on legit healthcare-adjacent roles), honest/em-dash-clean tailoring,
and the EMR/Excel gap-paragraph path.

## Known limitation

The sandbox this was built in has a non-functional headless LibreOffice, so the
one-page constraint is guaranteed structurally (tailoring adds no content to the
verified one-page base) rather than by rendering a PDF. To eyeball a `.docx`,
open it in Word/Google Docs or run `soffice --headless --convert-to pdf`.

## Scheduling

See `PIPELINE.md` for cron / Task Scheduler setup.
