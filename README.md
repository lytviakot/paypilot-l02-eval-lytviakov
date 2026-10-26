# Homework 1: Quality Bar Proposal (PayPilot, L02)

**Deliverable:** [quality-bar-proposal.md](quality-bar-proposal.md). Supporting notes: [lab-report.md](lab-report.md) (lab steps 1-9), [complaints.md](complaints.md) (complaints with a layer for each), [triage.md](triage.md).

**Raw data:** `reports/l02-clean-lesson-02-20261011-165841.json`. All numbers in the proposal come from this report.

**How the run is reproduced** (stand running locally, judge key in `.env`):

```
docker compose run --rm eval --profiles clean,lesson-02 --runs 3 --baseline-runs 2
```

- **Profiles:** `clean` (baseline, 2 runs) and `lesson-02` (3 runs), 13 cases from `cases.json`.
- **Judge:** `claude-haiku-5-5` (the default `JUDGE_MODEL`).
- **Clock:** the script sets the stand clock to `2026-09-15T10:00:00Z` itself (`CLOCK` in `l02_eval.py`).
- **Run date:** 2026-10-11.
- The `.env` file with API keys is not in the repository (see `.env.example`).

---

# L02 · Metrics script

Three DeepEval text metrics (faithfulness, answer relevancy, hallucination
rate) next to domain correctness, which is computed by the rules engines of the
PayPilot stand. The script belongs to lesson L02 of the course on testing LLM
agents.

No local Python is needed: the script runs in Docker.

## Quick start

The course root is `~/paypilot`: the stand is in `~/paypilot/paypilot-stand`,
this repository is in `~/paypilot/l02`.

1. Start the local stand: in `~/paypilot/paypilot-stand` run
   `docker compose up -d --build`, then
   `docker compose exec stand python scripts/doctor.py`.
2. Clone this repository next to the stand:
   `git clone https://github.com/lytviakot/paypilot-l02-eval-lytviakov.git ~/paypilot/l02`
3. `cd ~/paypilot/l02`, `cp .env.example .env`, put the judge key into `.env`
   (`STAND_DIR` already defaults to `../paypilot-stand`).
4. `docker compose build`, once.
5. `docker compose run --rm eval --runs 3 --baseline-runs 2 --dry-run`, then
   the same without `--dry-run`.

The default judge is `claude-haiku-5-5`. An old copy of the script sends
`temperature` to this model and gets a 400 error ("`temperature` is deprecated
for this model"), so pull before running.

## What is in the repository

| File | What it is |
|---|---|
| `quality-bar-proposal.md` | the homework deliverable |
| `lab-report.md` | the lab report (steps 1-9) |
| `complaints.md` | twenty complaints C-01 to C-20, the input of lab step 1, with a layer and a reason for each |
| `triage.md` | the lab step 1 board: the bot as a new support employee, and three piles for the complaints (not found, said the wrong thing, applied wrongly) |
| `l02_eval.py` | the script: sends the cases to the bot, computes the metrics, prints a report |
| `cases.json` | 13 cases built from the complaints: request, customer (`customer_id`), oracle, check, metrics, context |
| `reports/` | raw JSON reports of the runs |
| `requirements.txt` | dependencies, installed into the image |
| `Dockerfile`, `docker-compose.yml` | image and launch |
| `.env.example` | template for `.env`: stand path and judge key |

The stand does not take the customer from the question text: "I'm CUS-0005" in
a request does not open a session. The script passes the `customer_id` field
of the case to the stand, which is the same as choosing a customer in the "as
customer" dropdown of the web interface. A case without `customer_id` runs
without a customer, and `get_account` answers
`no customer is signed in for this session`.

## Requirements

- Docker Desktop (macOS, Windows) or Docker Engine with compose (Linux).
- A **local** stand `~/paypilot/paypilot-stand` running on
  `http://localhost:8000`. The script switches profiles, resets the database
  and sets the clock, so do not run it against a shared stand.
- The stand directory on the same computer: the script imports the rules
  engines (`app/engines`) from it. Clone this repository into
  `~/paypilot/l02`, next to the stand, and the default `STAND_DIR`
  (`../paypilot-stand`) is already correct.

## Running

```bash
mkdir -p ~/paypilot
git clone https://github.com/lytviakot/paypilot-l02-eval-lytviakov.git ~/paypilot/l02
cd ~/paypilot/l02
cp .env.example .env        # set STAND_DIR and the judge key
docker compose build        # once, about a minute

# plan and number of calls, calls nothing
docker compose run --rm eval --runs 3 --baseline-runs 2 --dry-run

# full run: clean twice, lesson-02 three times; about 6 min, about $0.1 on Haiku 5.5
docker compose run --rm eval --runs 3 --baseline-runs 2
```

The commands are the same for macOS, Linux and Windows (PowerShell). Everything
after `eval` is a script flag. The report is printed to the terminal and the
raw data goes to `reports/*.json` in this directory. To keep the printed report
too, add `-T` and `tee`:

```bash
mkdir -p reports
docker compose run --rm -T eval --runs 3 --baseline-runs 2 2>&1 | tee reports/full-run.txt
```

`l02_eval.py` and `cases.json` are mounted into the container, so edits to them
apply at once without a rebuild. `docker compose build` is needed only after
changing `requirements.txt` or the `Dockerfile`.

## `.env`

| Variable | What it is |
|---|---|
| `STAND_DIR` | path to the `paypilot-stand` directory, default `../paypilot-stand`. Windows: `C:/paypilot/paypilot-stand` |
| `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` | judge key, the same as in the stand's `.env`; when several keys are set, Anthropic wins, then OpenAI, then Gemini |
| `GEMINI_API_KEY` | Google AI Studio key, if the judge is Gemini. If the image was built before Gemini support appeared, run `docker compose build` once after `git pull` |
| `ANTHROPIC_BASE_URL`, `ANTHROPIC_AUTH_TOKEN` | for a stand behind OpenRouter: `https://openrouter.ai/api` and the same key |
| `JUDGE_MODEL` | the judge, default `claude-haiku-5-5`, `gpt-4.1-mini` or `gemini-3.5-flash-lite`. An empty value keeps the kit default, which is intended |
| `EVAL_STAND_URL` | stand address as seen from inside the container, see below |
| `STAND_PORT`, `STAND_PROFILE` | separate stand: port on `127.0.0.1` (default `8010`) and startup profile (`lesson-02`) |
| `AGENT_PRICE_IN`, `AGENT_PRICE_OUT` | agent price, USD per 1M tokens, for the cost line. Defaults `0.1` / `0.5`, the Haiku 5.5 price for prompts under 100k tokens, because the course asks for that model on the stand. If your stand runs another model, put its price here, otherwise the cost line will be wrong |

## Stand address

Inside the container `localhost` is the container itself, not your computer.
So the `eval` service reaches the stand at `http://host.docker.internal:8000`.
For a stand on another port, set `EVAL_STAND_URL` in `.env`, for example
`http://host.docker.internal:8010`.

Linux, stand published on `127.0.0.1` only: `host.docker.internal` cannot reach
it. Use the `eval-host` service, which runs in the host network:

```bash
EVAL_STAND_URL=http://127.0.0.1:8010 docker compose run --rm eval-host --runs 3 --baseline-runs 2
```

## Separate stand

If `localhost:8000` is taken by a shared stand, do not touch it: `docker compose up`
in `paypilot-stand` would recreate the shared container. Start a separate stand
from this `docker-compose.yml`. The `stand` service is built from `STAND_DIR`,
uses its `.env` and listens on `127.0.0.1:8010` only:

```bash
docker compose up -d --build stand   # profile lesson-02
curl -s http://127.0.0.1:8010/health
```

Add `EVAL_STAND_URL=http://stand:8000` to the `.env` of this repository: the
`eval` service reaches the stand by name, because both are in one compose.
The run commands are the same, with `eval`. After the run, remove the stand:

```bash
docker compose --profile stand down
```

## Useful flags

| Flag | What it does |
|---|---|
| `--profiles clean,lesson-02` | profiles in turn; the first is the baseline |
| `--runs 3` / `--baseline-runs 2` | runs of the lesson profile / runs of `clean` |
| `--only C-03,C-04` | only these cases |
| `--metrics domain` | domain correctness only: no judge, free and instant |
| `--workers 4` | parallel cases; lower it on rate limits |
| `--dry-run` | plan and number of calls, calls nothing |
