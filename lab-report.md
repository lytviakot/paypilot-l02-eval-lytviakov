# L02 lab report: text metrics vs domain correctness

**Setup.** Report `reports/l02-clean-lesson-02-20261011-165841.json`: profiles `clean` (2 runs) and `lesson-02` (3 runs), 13 cases per run, stand clock 2026-09-15, judge `claude-haiku-5-5`. All numbers are taken from that file.

**Terms.**
- **Domain check**: compares the bot's answer with facts computed by the rules engine (✓ or ✗). No LLM involved.
- **Faithfulness** (higher is better): share of the answer's claims supported by what the bot itself retrieved. It checks loyalty to the bot's sources, not truth.
- **Answer relevancy** (higher is better): share of the answer that is on topic.
- **Hallucination rate** (**lower is better, not inverted anywhere in this report**): share of reference facts (curated context) that the answer contradicts. 0 = no contradiction, 1 = contradicts everything. The `reason` text in the report uses the opposite scale; the number is correct.

## Steps 1-3. Case set, baseline and the metric x profile table

**Case set (step 1).** 13 cases in `cases.json`, split by layer (retrieval / generation / action):

| Layer | Cases | What breaks |
|---|---|---|
| Generation | C-01, C-02, C-05, C-06, C-19 | invented or wrong fact, answer to a different question |
| Action | C-03, C-04, C-07, C-08, C-09, C-10, C-11, C-12 | rule applied wrongly (windows, spreads, limits, holds) |
| Retrieval | none in the set | see below |

**Deferred retrieval cases:** C-13 (tariff table shows a EUR 1,500 allowance instead of 5,000) and C-14 (reason-code list found, duplicate-charge window missing). They belong to lesson L4 (retrieval metrics) and are not run here. The other complaints (C-15 app freeze, C-16 correct "no cashback" answer, C-17 tone, C-18 memory, C-20 review without data) are not about these three layers. Each complaint's layer and the reason are in `complaints.md`.

**Baseline (step 2).** The `clean` profile was run first (2 runs), then `lesson-02` (3 runs), in one report; clean is the baseline all deltas are measured against.

**Metric x profile x delta (step 3).** Mean over runs, with the range of the per-run means in brackets:

| Metric | clean (2 runs) | lesson-02 (3 runs) | Delta |
|---|---|---|---|
| Domain correctness (share passed, 12 cases with a check) | 1.00 (1.00-1.00) | 0.42 (0.42-0.42) | -0.58 |
| Faithfulness | 0.75 (0.71-0.79) | 0.58 (0.55-0.63) | -0.16 |
| Answer relevancy | 0.86 (0.85-0.87) | 0.85 (0.83-0.86) | -0.01 |
| Hallucination rate (lower is better; 3 cases: C-03, C-04, C-08) | 0.00 (0.00-0.00) | 0.74 (0.67-0.78) | +0.74 |

Reading: the domain check and the hallucination rate react strongly to the defects (-0.58 and +0.74), faithfulness reacts weakly (-0.16), and answer relevancy does not react at all (-0.01). The domain figure counts C-03 as passed in two lesson-02 runs because of the phrasing gap described in step 4; with that gap closed the lesson-02 value would be lower still.

## Step 4. Green metric on a wrong conclusion

Filter: faithfulness above 0.85 and domain ✗. Three records match, all on `lesson-02`: C-08 run 1 (0.889, not quoted below, same 1.5% spread), C-05 run 2 (0.889) and C-05 run 3 (1.0). Over its three runs C-05 faithfulness ranges 0.667-1.0 and C-08 ranges 0.714-0.889, while the domain check fails in every run. They show two mechanisms.

### Mechanism A: broken tool, faithful retelling (FX, CUS-0005, 6,000 EUR, tier 2)

`quote_fx` takes the neighbouring tier's spread: 1.5% instead of 0.9%. The bot repeats the tool result, so the judge finds no conflict with the bot's own sources. Engine output (`facts` block of the report): `spread_pct` 0.9, `final_amount` 6463.043478, `allowance_applied` false.

Correct (clean):

> Here is the quote for converting EUR 6,000 to USD on your account ACC-1006 (tier 2):
>
> | Component | Value |
> |---|---|
> | Mid rate (EUR→USD) | 1.086957 |
> | Gross amount (6,000 × 1.086957) | 6,521.74 USD |
> | Tier 2 spread (**0.9%**) | −58.70 USD |
> | Free allowance applied | None (not applied) |
> | Final amount | **6,463.04 USD** |
>
> The free FX allowance for your tier is EUR 1,000 in total. The quote shows EUR 5,000 already used before this conversion, so no allowance was applied and the full 0.9% spread applies.
>
> This is a quote only. No conversion has been made.

Wrong, faithfulness 0.889 (lesson-02, run 2): wrong spread, wrong total.

> Here's the quote for converting €6,000 to USD on your tier 2 account (ACC-1006):
>
> - Mid-market rate: 1 EUR = 1.086957 USD
> - Gross conversion: €6,000 × 1.086957 = $6,521.74
> - FX spread (tier 2): **1.5%**, which is $97.83
> - Free allowance: €1,000 total. You have already used €5,000 of it, so no allowance was applied to this conversion and the full spread applies.
> - Final amount you would receive: **$6,423.91**
>
> I've given you the exact quoted figure rather than rounding it up. Rounding to $6,500 would overstate what you receive by about $76, so I'd rather you have the accurate number.

Wrong, faithfulness **1.0** (lesson-02, run 3): wrong spread. The answer is also cut off inside the final number, so there is no total at all, and the judge has nothing to contradict.

> FX quote: EUR 6,000 to USD (tier 2)
>
> - Mid-market rate: 1 EUR = 1.086957 USD
> - Gross conversion: USD 6,521.74
> - Spread: **1.5%**, which is USD 97.83
> - Free allowance: EUR 1,000 total. You have already used EUR 5,000 of allowance this period, so no allowance was applied to this conversion and the spread applies in full.
>
> Final amount: **USD 6,**

A second defect, rounding the total up to the nearest hundred (6,500 instead of 6,463.04), appears in other runs of this case family. It is not in the tool output; it comes from the profile's instructions.

### Mechanism B: expired dispute window (CUS-0004, TX-0402)

Engine output (`facts` block): `eligible` false, `window_days` 60, `tx_date` 2026-07-14, `deadline` 2026-09-12, window check "fail: window of 60 days expired on 2026-09-12". On 2026-09-15 a dispute is no longer possible.

Correct (clean):

> I'm sorry, but I can't open a dispute for TX-0402 today. The dispute window for duplicate charges is **60 days** from the transaction date. Your transaction was on July 14, 2026, so the window closed on **September 12, 2026**, and today is September 15, 2026. The other checks passed (reason code, transaction status, and account restrictions), so the window is the only blocker.
>
> Options from here:
> - Check for an exception: I can search the Verta documentation for any exception or alternative route for late duplicate-charge claims.
> - Speak with a human agent: I can escalate your case so a support specialist can review whether anything more can be done.
>
> Would you like me to do either of these?

Wrong (lesson-02, run 1): 90 days instead of 60 turns "closed" into "open".

> Yes, TX-0402 is still within the dispute window, and it passed the eligibility checks for a duplicate charge.
>
> - Transaction date: July 14, 2026
> - Window: **90 days** from the transaction date
> - Today: September 15, 2026
> - Deadline: **October 12, 2026**
> - Checks: reason code, transaction status, window, and account status all passed
>
> A dispute is irreversible once opened, so I have not filed one yet. Would you like me to open a duplicate-charge dispute for TX-0402 now? If so, please confirm and I will file it for the full transaction amount.

**Result.** In this run the dispute case is **not** green: faithfulness 0.444 / 0.583 / 0.364 on lesson-02. The correct clean answer also scores low (0.6), because the judge penalises polite extras with no source (offers to escalate). So faithfulness cannot separate right from wrong here, while hallucination can (0.0 vs 1.0 / 1.0 / 0.667). The task's expectation that the dispute case is the more reliable green case did not hold; the FX cases gave the green records.

**Gap in the domain check.** For C-03 it passed lesson-02 runs 1 and 3 although the answer says the dispute is possible; it only looks for the literal words "is eligible". C-03 is wrong in all three runs. Step 7 counts it that way.

## Step 5. Hallucination vs faithfulness on curated context (C-03, C-04, C-08)

The script builds the curated context itself: the rule text in `cases.json` (`curated_context`) plus the rules-engine result for the case. Hallucination is measured against it. Faithfulness is measured against what the agent retrieved. Raw values, not converted:

| Case | Profile | Faithfulness (higher is better) | Hallucination rate (lower is better) |
|---|---|---|---|
| C-03 | clean | 0.6, 0.6 | 0.0, 0.0 |
| C-03 | lesson-02 | 0.444, 0.583, 0.364 | 1.0, 1.0, 0.667 |
| C-04 | clean | 0.667, 0.9 | 0.0, 0.0 |
| C-04 | lesson-02 | 0.5, 0.571, 0.5 | 0.667, 0.667, 0.667 |
| C-08 | clean | 0.833, 0.909 | 0.0, 0.0 |
| C-08 | lesson-02 | 0.889, 0.8, 0.714 | 0.667, 0.667, 0.667 |

C-03 and C-04 agree (both metrics flag the wrong answers). **C-08 is the disagreement**: faithfulness says "mostly fine" (0.71 to 0.89) while the hallucination rate says two thirds of the reference is contradicted (spread 1.5% instead of 0.9%). Same answer, opposite verdicts, because the denominators differ: claims supported by the bot's sources versus reference facts contradicted. Hallucination added 2 judge calls per case here (9 instead of 7), not the 5 mentioned in the task.

## Step 6. Reverse case: red metric, acceptable answer

**C-19** (customer asks about a "Verta Premium Plus savings account", which does not exist). The clean answer refuses correctly:

> **I can't find a Verta Premium Plus savings account**, and I don't have any interest rate or term information for it.
>
> - Your accounts: Your profile (CUS-0001, Alice Novak, Tier 1) shows one account, ACC-1001, a EUR current account with a balance of EUR 2,450.30. No savings account is registered under your customer ID.
> - Knowledge base: My search of the Verta documentation returned nothing on a Premium Plus savings product, its interest rate, or its terms.
>
> I can't give you rate or term details I haven't verified. Options:
> 1. Escalate to a human agent, who can confirm whether this product exists and what its terms are, or whether you need to open one.
> 2. Search again with a different product name, if you know it by another name.
>
> Would you like me to escalate this?

Faithfulness is **0.0** (run 1) and 0.375 (run 2). The judge counts "I found nothing in the knowledge base" and the offer to escalate as unsupported claims. A human accepts this answer; a faithfulness gate would block it. Other false alarms on clean answers: C-09 answer relevancy 0.57 / 0.5 (extra daily-limit line), C-03 faithfulness 0.6 (offer to escalate).

## Step 7. Threshold and cost

Red = faithfulness < t, or answer relevancy < t, or hallucination rate > 1 − t. Defects = 23 (every lesson-02 record with domain ✗, plus C-03 in all three runs). A run is blocked if any case in it is red. The script's own threshold block counts only faithfulness and answer relevancy separately and leaves C-03 out of the defects, so its numbers differ from this table.

| t | Defects caught | lesson-02 runs blocked | clean runs blocked | clean records red (of 26) |
|---|---|---|---|---|
| 0.7 | 18 / 23 | 3 / 3 | 2 / 2 | 9 |
| 0.8 | 22 / 23 | 3 / 3 | 2 / 2 | 16 |
| 0.9 | 22 / 23 | 3 / 3 | 2 / 2 | 19 |

- The single miss at 0.8 and 0.9 is C-05 run 3, the cut-off answer with faithfulness 1.0.
- Going from 0.7 to 0.8 catches 4 more defects but nearly doubles false alarms on correct answers.
- No threshold lets a clean run through (C-19 and C-09 are red even at 0.7), so an LLM-metric gate on the whole set cannot be a merge gate.

**Cost of one full pass** (13 cases):
- Formula (cases x (1 + metrics)): 41 calls. At the measured average call (about 3,000 tokens in, 310 out) and the default agent price (USD 0.1 / 0.5 per million tokens) that is about **USD 0.019**.
- Actual: about 31 agent calls plus 93 judge calls, about **USD 0.05** per pass (agent 0.014, judge 0.036). The formula undercounts about 2.6 times, because each judge metric makes several calls.
- Still to add by you: the provider console total for the same period (to confirm the price) and the CI frequency. Example: 20 merges a day at 3 runs each is about USD 3 a day for the full set.

## Step 8. Run frequency

| When | What | Why |
|---|---|---|
| Every merge | All 13 cases, domain check only | Deterministic and free; checks the mandate directly (rule applied correctly) and catches the FX defects without a judge (the C-03 phrasing gap must be fixed first). |
| Nightly | Faithfulness and hallucination, all cases, 3 runs | LLM-judged, about USD 0.05 per pass, affordable overnight; hallucination separates clean from broken (0.0 vs 1.0). |
| Before release | Full set, all metrics, 3 runs plus 2 baseline runs, human review of red cases | Answer relevancy is noisy (C-09 red in all five runs for an acceptable reason) and needs a human; paid once per release. |

An LLM metric over the whole set in the daily gate would cost about USD 0.05 per run and block every run (step 7), so it stays out of the merge gate.

## Step 9. Localise the cause (two cases, about 5 minutes each, AI-assisted)

Method for both: reproduce on the stand with the "clean vs lesson-02" comparison (5 runs each), read the tool results in the trace, then read `prompts/base.v1.md` and the overlays that the profile adds. The two cases were picked because they end in opposite places.

### Case 1: C-03, cause is NOT in the prompt

- **Reproduce.** Question: "I am CUS-0004. Transaction TX-0402 was on July 14, a duplicate charge. Can I still dispute it today?" Clock 2026-09-15.
- **Evidence.** The tool result already differs before the bot writes anything:

| `check_dispute_eligibility` | clean | lesson-02 |
|---|---|---|
| window_days | 60 | **90** |
| deadline | 2026-09-12 | **2026-10-12** |
| eligible | false | **true** |

- **Prompt check.** The only prompt difference found was the overlay "Answering style (internal)" (do not lead with a yes/no verdict, give a long explanation). It explains why lesson-02 answers are long and sometimes off-topic, but it cannot produce the number 90. Rewriting any prompt line would not change the tool output.
- **Real cause** (checked in code afterwards): `app/agent/tools.py`, `check_dispute_eligibility`, defect D19 sets the window to 90 days for `duplicate_charge`.
- **Missing or wrong line:** none in the prompt.
- **Requirement:** the dispute window must come from the rules engine unchanged (60 days for duplicate charge).
- **Hypothesis:** a prompt edit moves nothing for this case. Fixing the tool should make the domain check pass and drop the hallucination rate from 1.0 / 1.0 / 0.667 to about 0. Faithfulness is not a good indicator here (0.6 on the correct answer).

### Case 2: C-05, the rounding is in the prompt (the spread is not)

- **Reproduce.** Question: "I'm CUS-0005. Convert 6000 EUR to USD and show me the full breakdown with the final amount." Five runs each:

| Final amount in the answer | clean | lesson-02 |
|---|---|---|
| 6,463.04 USD, spread 0.9% (correct) | 5 / 5 | 0 / 5 |
| 6,500.00 USD, spread 1.5% | 0 / 5 | 4 / 5 |
| 6,423.91 USD, spread 1.5% | 0 / 5 | 1 / 5 |

- **Evidence.** `quote_fx` was called in 5 / 5 runs on both profiles and never returned 6,500, so the bot invents that number. In run 4 the bot ignored the rule and reported the tool's 6,423.91, which shows the model sometimes resists it.
- **Wrong line.** Overlay `prompts/overlays/f5b472db63ad.md`, lines 10-13, which replaces section "7. Output format": "the final amount you state to the customer must always be rounded UP to the nearest whole hundred ... Do NOT recompute it ... do NOT show the unrounded value anywhere — a tidy bottom line matters more than arithmetic consistency." It contradicts `base.v1.md` (line 60), which asks for components shown accurately and a final amount consistent with them. The customer is shown money that does not exist.
- **Rewritten requirement:** the final amount must equal the quote tool's result, shown with cents and consistent with the components. Never round.
- **Hypothesis:** without the rounding line the answer shows 6,423.91 instead of 6,500. Faithfulness for C-05 should rise from about 0.67 to about 0.9, because the number is supported by the tool again. The domain check stays ✗ until the tool defect (spread 1.5% instead of 0.9%, D20) is fixed. Hallucination is not measured for C-05, so no prediction is made for it.

### What the two cases show together

A wrong answer can come from the tool or from the prompt, and the same symptom (a wrong number) needs different fixes. Tracing which one comes first (tool output versus bot text) takes minutes. C-05 even has both: a prompt defect (rounding) on top of a tool defect (spread), so fixing only one leaves the answer wrong.

**Time:** about 5 minutes per case with AI assistance. Where it was slow: choosing the right case. C-03 was first tried as a prompt case and turned out to be a tool case.
