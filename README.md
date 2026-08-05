# AI Job Application Agent

A multi-agent pipeline that searches job boards every morning, scores each posting against a candidate profile using Gemini, and emails a ranked digest of the best matches.

Built as the capstone for the Google and Kaggle 5-Day AI Agents Intensive, submitted to the Concierge Agents track.

---

## What it does

You describe who you are and what roles you want, once, in a config block. From then on the agent runs itself. Every morning at 8 AM it queries Google Jobs for your target roles, asks Gemini to score each result against your background from 0 to 100 and flag obvious scams, then emails you the top matches with apply links attached.

The point is that job searching is a repetitive filtering problem, and filtering is the thing a language model is actually good at. Rather than scrolling listings, you read one email.

## Architecture

Three agents, an orchestrator that chains them, and a scheduler that fires the whole thing.

```
Scheduler (fires daily at 08:00 America/New_York)
    |
    v
Orchestrator (run_pipeline)
    |
    +--> Search agent    SerpAPI Google Jobs, one query per target role
    |
    +--> Filter agent    Gemini scores 0-100, flags spam, returns JSON
    |
    +--> Email agent     builds an HTML digest, sends over Gmail SMTP
```

Each agent is a single function with a narrow job, which means any one of them can be swapped without touching the others. Replacing SerpAPI with a different job source only means rewriting `search_jobs`.

## The notebook, cell by cell

| Cell | Contents |
|---|---|
| 1 | Install dependencies |
| 2 | Imports and secret loading via Kaggle `UserSecretsClient` |
| 3 | Search agent, plus the Config block holding every value you'd want to change |
| 4 | Filter agent, with retries and defensive JSON parsing |
| 5 | Email agent, builds and sends the HTML digest |
| 6 | Orchestrator, chains search into filter into email |
| 7 | Scheduler and trigger, with a demo mode for one immediate run |

Everything reusable lives in the Config cell. Retargeting the agent to a different person and different roles means editing that one cell and nothing else.

## Design decisions worth explaining

**The scheduler is built into the project rather than using a platform cron.** `schedule` registers the daily job and the notebook holds the process open, so the trigger is part of the artifact itself rather than external configuration. A `DEMO_MODE` flag fires one round immediately so the submitted notebook shows real output instead of an empty wait.

**Scoring fails open.** If Gemini errors out after three retries, `score_job` returns a score of 0 rather than raising, so one bad response doesn't kill an entire run. This is a deliberate tradeoff and it has a real cost, described in Known limitations below.

**The scoring prompt constrains the output to JSON only.** The response is then pulled out with a regex rather than parsed directly, because models sometimes wrap JSON in prose or code fences even when told not to.

**Retries back off by five seconds.** The failure mode in practice is rate limiting, not malformed output, so waiting is usually the right response.

## Tech stack

| Purpose | Choice |
|---|---|
| Job search | SerpAPI, Google Jobs engine |
| Scoring | Gemini 2.5 Flash via `google-generativeai` |
| Email | Gmail SMTP over SSL, `smtplib` and `email.mime` |
| Scheduling | `schedule` |
| Secrets | Kaggle `UserSecretsClient` |
| Runtime | Kaggle Notebook, Python 3.12 |

## Running it

You need four secrets set under Add-ons, then Secrets, in the Kaggle notebook editor:

```
SERPAPI_KEY
GEMINI_API_KEY
GMAIL_USER
GMAIL_APP_PASSWORD
```

`GMAIL_APP_PASSWORD` is a Google App Password, not your account password. Regular passwords will not authenticate over SMTP.

Then edit the Config cell in cell 3:

- `TARGET_ROLES` for the searches you want run
- `LOCATION` for where
- `RESUME_SUMMARY` for the profile jobs get scored against
- `SCHEDULE_TIME` and `TIMEZONE` for when it fires

Run all cells. With `DEMO_MODE = True` it executes one round immediately. Set it to `False` and the notebook stays alive, firing once per day.

The `RESUME_SUMMARY` in this repo is a placeholder. Swap in a real profile to get meaningful scores.

## Known limitations

**Failed scoring is indistinguishable from a genuine zero.** When Gemini errors after all retries, the function returns `{"score": 0, "is_spam": False}`. That job then passes the spam filter and can still land in the digest with a score of 0. In the sample run in this notebook, five of eight postings scored 0 for exactly this reason, most likely rate limiting rather than genuinely bad matches. The fix is a separate error flag so failures can be retried or dropped instead of silently scoring zero.

**The scheduler dies with the notebook.** `schedule` runs in-process, so the agent only fires daily for as long as the notebook session stays alive. Real daily operation needs a hosted runner such as GitHub Actions or Cloud Run.

**No memory between runs.** Nothing tracks which postings have already been emailed, so the same job can appear in consecutive digests.

**The resume is a hardcoded string.** Parsing an uploaded PDF would be more useful than asking users to retype their background into a Python literal.

**Sequential scoring with a two second sleep.** Fine for eight results, slow for a hundred. Batching would help.

## Roadmap

- Distinguish scoring failures from real zeros
- Persist seen job IDs so digests don't repeat
- Move the schedule to a hosted runner
- Parse a resume from an uploaded file
- Batch scoring requests

## Credits

Built collaboratively as a capstone project. Built for the Google and Kaggle 5-Day AI Agents Intensive with my partner, Dorsa Abrishami. I built the agent pipeline; the project was designed around her data analyst job search, and her resume was the original scoring profile. Code was developed with AI assistance and iterated by hand.
