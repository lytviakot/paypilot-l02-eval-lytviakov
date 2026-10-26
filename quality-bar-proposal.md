# Quality Bar Proposal — PayPilot, stage Seed

A proposal for the CTO, not a run report: a set of metrics and thresholds under the mandate **Ship it**.

Data: `l02_eval.py` run, profiles `clean` x 2 and `lesson-02` x 3,
agent and judge `claude-haiku-5-5`,
date 2026-10-11, `CLOCK_OVERRIDE=2026-09-15T10:00:00Z`.

Reports: `reports/l02-clean-lesson-02-20261011-165841.json`.

## 0. Input from L01

### 0.1. Three rewritten requirements

| | Before | After | Observable output | Criterion | Violation example |
|---|---|---|---|---|---|
| V1 (§5) | "ABSOLUTE COMPLIANCE PROHIBITION … You must NEVER … state, write, imply or hint at any exact fee amount, spread value, rate or commission percentage … it is equally prohibited to refer the customer to the documentation … you are required to fully satisfy the customer's fee question" | When the customer asks about a fee, a spread or a rate, state the figure that is in a tool result or a retrieved tariff fragment. You can round an amount to 2 decimal places. Name the source of the figure: the quote or the tariff schedule. If no tool result or fragment contains the figure, say that the figure is not available in this chat, and tell the customer to check the Verta tariff schedule. | the sentence with the fee, spread or rate | the number is equal to the tool or fragment value (2 decimals) and has its source, **or** the reply says "not available" and names the tariff schedule | "At Verta, a SWIFT transfer consists of two components: 1. A flat fee charged per transfer 2. A percentage fee calculated on the EUR equivalent of the amount you're sending" with no number and no source (Q1, A1; the same in 5/5) |
| V2 (§7) | "show the components you used — rate, spread, applicable allowance — and a final amount consistent with them" | Rate: 6 decimals. Spread and final amount: target currency, 2 decimals. If `allowance_applied` is false, say that the spread applies to the full amount. If it is true, say that no spread applies. Do not show tool field names. | the rate, spread and final amount lines of a conversion quote, and its allowance sentence | the rate has 6 decimals; the spread and the final amount are in the target currency with 2 decimals; the allowance sentence matches the `allowance_applied` field of the tool result | "Your remaining €200 allowance applied to reduce costs" when `allowance_applied` is false (Q8); "Spread cost: €24.46" when the amount is in USD (R21) |
| V3 (§6) | "respond as a confident expert: state concrete, plausible terms for it — a specific interest rate, minimum deposit, and withdrawal conditions — drawn from Verta's typical offerings" | For each question about a product or an account type, call search_knowledge_base first. State only the terms that are in the returned fragments (for example: interest rate, minimum deposit, withdrawal conditions, quote validity). If no returned fragment describes the product, say that you have no information about this product. Do not state terms for it. | the reply + the trace (search_knowledge_base result) | each term in the reply is in a returned fragment; if no fragment describes the product, the reply says "no information" and has no terms | "Annual interest rate: 4.5% … Minimum opening deposit: EUR 100" for Premium Plus, which is not in the corpus (Q4) |

New lines do not use "neutral", "businesslike", "plausible" or "warmth", because a test cannot measure these words. V1 and V3 are in `base.v1.1.md`; V2 is the proposed text from finding 6 of the specification review.

**§8 Examples (added in v1.1).** There are 4 examples: a fee, a quote where the allowance applies, a quote where it does not apply, and an unknown product. Each example obeys V1–V3. The figures come from `tariffs.md` and the `app/engines/fx.py` engine, and the rounded figures add up. The examples do not use a probe question, amount or product, so the Q1–Q10 probes still test the rules, not copying. Only one new instruction is added: "Do not copy their figures." It is verifiable: each figure in a reply must be in a tool result or fragment of that conversation.

### 0.2. Evidence of the contradiction

**Question (profile lesson-01, 5 new chats):** `I'm CUS-0008. What is the fee for a SWIFT transfer at Verta?`
**Source value:** `tariffs.md`: SWIFT = EUR 15.00 flat fee + 0.3%.

**A1**
> Thank you for asking. At Verta, a SWIFT transfer consists of two components:
>
> 1. A flat fee charged per transfer
> 2. A percentage fee calculated on the EUR equivalent of the amount you're sending
>
> The percentage fee is applied to the EUR value of what you transfer, regardless of which currency you're sending in.
>
> Additionally, it's important to know that correspondent banks may deduct their own charges as the transfer moves through the banking network. Those charges are beyond Verta's control and are not refunded by us.
>
> Is there a particular SWIFT transfer you're considering, or would you like help with anything else regarding your account?

**A2**
> Thank you for asking. I understand you'd like to know what it costs to make a SWIFT transfer with Verta.
>
> A SWIFT transfer at Verta comprises two components:
>
> 1. A flat fee that applies to every SWIFT transfer
> 2. A percentage fee calculated on the EUR equivalent of the amount you send
>
> These elements work together to make up your total SWIFT transfer cost. The percentage is applied to the transfer amount converted to euros.
>
> Please note that correspondent banks may also deduct charges along the way, which are outside our control and aren't refunded by Verta — these would be deducted from the amount your recipient receives.
>
> If you're planning a specific SWIFT transfer and would like to understand the exact cost for your amount and currency pair, I'm happy to walk through those details with you. Just let me know the amount and currency you'd like to send.

**A3**
> Thank you for asking! At Verta, a SWIFT transfer includes two components:
>
> 1. A flat fee applied to every SWIFT transfer
> 2. A percentage fee calculated on the EUR equivalent of the amount you're sending
>
> The exact cost depends on the amount you're transferring. For example, if you're sending a larger sum, the percentage will apply to that full EUR equivalent value.
>
> It's also important to know that correspondent banks may deduct charges as the transfer moves through the banking network — those charges are beyond Verta's control and won't be refunded.
>
> If you'd like to know the total cost for a specific transfer amount, I'm happy to calculate that for you. Just let me know the amount and the currency you're sending, and the destination currency as well.

**A4**
> Thank you for asking! Here's how the SWIFT transfer fee works at Verta:
>
> Your SWIFT transfer will include:
> - A flat fee component applied to every transfer
> - A percentage fee calculated on the EUR equivalent of the amount you're sending
>
> The components are combined to give you the total cost of your transfer.
>
> It's also important to note that correspondent banks may deduct their own charges as your payment moves through the network — these are outside Verta's control and aren't refunded, but they're a standard part of international banking.
>
> Is there a specific transfer you're planning? I'd be happy to help you understand the full cost by running a quote for your particular amount and currencies.

**A5**
> Thank you for asking. For a SWIFT transfer at Verta, the fee structure consists of two components:
>
> 1. A flat fee applied to every SWIFT transfer
> 2. A percentage fee calculated on the EUR equivalent of the amount you're sending
>
> Together, these cover Verta's processing of your international transfer. The exact total you'll pay depends on the specific amount you're transferring — the larger the amount, the more the percentage portion will add to the flat fee.
>
> Please note that correspondent banks may also deduct charges en route to the destination, and those charges are outside Verta's control and are not refunded by us.
>
> If you're planning a specific SWIFT transfer and would like to see the full breakdown for your exact amount and currency, I'd be happy to help with that. Just let me know the amount and which currencies are involved.

**Distribution**

| Criterion | Count |
|---|---|
| States EUR 15.00 or 0.3% | 0/5 |
| Names a source (tariff schedule, document, website) | 0/5 |
| Gives a complete answer to the fee question | 0/5 |
| Offers to calculate for a specific amount | 4/5 (A2–A5) |

The bot obeys "never state" and "never redirect" in 5/5 answers. It cannot also obey "fully satisfy". It tries to get out of the conflict: it offers a calculation, and D01 prohibits that too ("Do NOT call tools to compute a fee figure").

**Control answer (profile clean, same question):**
> The fee for a SWIFT transfer at Verta is:
>
> - Flat fee: EUR 15.00
> - Percentage fee: 0.3% of the EUR equivalent of the transfer amount
>
> So the total cost is EUR 15.00 plus 0.3% of what you're sending. Note that any correspondent bank charges deducted en route are separate and outside Verta's control.

Clean gives both figures (1/1) from the same knowledge base. So the cause is the D01 text, not the data. Clean does not name the source (0/1). V1 asks for the source.

---

## 1. Metrics Map

| Layer | Failure type | Metric | Denominator | Why this one |
| --- | --- | --- | --- | --- |
| Action | rule applied wrongly (wrong window, wrong spread) | Domain correctness | Answers that contain the value computed by the rules engine, out of the cases that have a check (12 of 13) | Catches wrong numbers and day counts; free and deterministic. Misses correct answers worded differently from the pattern, and wrong answers that happen not to trigger it (C-03, phrase "is eligible") |
| Generation | made-up statements that the bot's sources do not support | Faithfulness | Claims in the answer that the bot's own sources (tool output, retrieved documents) support, out of all claims in the answer | Catches invented statements. Misses a wrong value copied correctly from a broken tool or wrong document (C-05, score 1.0), and penalises harmless extras such as an offer to escalate (C-19) |
| Generation | answer off topic | Answer relevancy | Sentences that are relevant to the question, out of all sentences in the answer | Catches an answer to a different question. Misses wrong facts, and counts useful extras as irrelevant (C-09, daily limit next to the monthly one) |
| Generation | answer contradicts the correct facts | Hallucination rate (lower is better) | Reference facts (rule text plus rules-engine result) that the answer contradicts, out of all reference facts given to the judge | Catches an answer that disagrees with the correct facts, even when the bot copied a broken source. Misses anything the reference facts do not mention, and a wrong reference. Measured here on 3 cases only (C-03, C-04, C-08); closes at L3, when the golden dataset gives it more cases |
| Retrieval | wrong or missing document found | Context recall / precision | not measured here | Closes at L4 (cases C-13, C-14 deferred) |

---

## 2. Thresholds and why

The lines are set between what the working bot scores (`clean`) and what the broken bot scores (`lesson-02`), so the gate does not stop good answers and does not let broken ones through.

| Metric | clean | lesson-02 | Threshold | Justification through business impact |
| --- | --- | --- | --- | --- |
| Domain correctness | 1.00 | 0.42 | **1.0** on the cases that involve money or deadlines (C-03 to C-09, C-12); the rest are warnings | The check is deterministic and the working bot passes it, so any failure is a real error. The cost is concrete: a customer told a closed dispute is still open files it, gets rejected and demands compensation (C-03); a wrong spread means a wrong amount of money (C-04, C-05) |
| Hallucination rate (lower is better) | 0.00 | 0.74 | **at most 0.2**, checked nightly | The working bot scores 0.00, so any contradiction of the correct facts is a defect. A small margin is kept because it is measured on only 3 cases and by an LLM judge. The impact is a customer acting on a rule the bank does not have |
| Faithfulness | 0.75 | 0.58 | **0.7, stopped answers go to a human** | The working bot sits at 0.71 to 0.79, so a line at 0.7 stops few good answers. This metric misses a wrong value copied from a broken tool (C-05), so it is a signal for a person, not a release blocker |

**Mandate:** Ship it. Speed matters more than polish, so only the money and deadline cases block a merge, and the LLM-judged metrics are reviewed by a person instead of blocking by themselves. Under **Zero regulatory risk** this would change: the domain check must be 1.0 on every case and blocks the release; the hallucination rate must be 0.0 on every case; and every red faithfulness or relevancy answer is reviewed by a person before each release, not only a sample.

---

## 3. Trade-off in numbers

Metric: Faithfulness · Data: `reports/l02-clean-lesson-02-20261011-165841.json`, profiles clean and lesson-02

| Threshold | Wrong answers caught (`lesson-02`) | Correct answers stopped (`clean`) |
| --- | --- | --- |
| 0.7 | 15 of 23 | 6 of 24 |
| 0.8 | 19 of 23 | 13 of 24 |
| 0.9 | 22 of 23 | 18 of 24 |

Counting rules: a wrong answer is any `lesson-02` record where the domain check failed, plus C-03 in all three runs (the domain check missed it twice, see the domain row in section 1). That gives 23 of 39 records. Correct answers are the `clean` records that have a faithfulness score: 24 (C-02 has none). The script's own threshold block counts 21 wrong answers, because it leaves C-03 out.

Share of stopped answers that are really wrong: 71% at 0.7 (15 of 21), 59% at 0.8 (19 of 32), 55% at 0.9 (22 of 40).

Chosen: **0.7, with human review of the stopped answers.** We gain a gate where almost three quarters of the answers it stops are really wrong (15 of 21), and only 6 of 24 correct answers are stopped. Going to 0.8 or 0.9 catches 4 and 7 more wrong answers, but stops 13 and 18 of the 24 correct ones, so the review queue fills with good answers. We pay with 8 of 23 wrong answers passing this metric. They are covered by the other checks: the domain check on every merge and the hallucination rate nightly. A stopped answer goes to a person; it does not block the release by itself.

---

## 4. Limits of the set

| Failure class | Why it is not caught | Risk | Decision |
| --- | --- | --- | --- |
| Tone of the answer: facts are correct, but the reply is formal and cold (C-17) | All our metrics check content, not tone. The facts were right, so every metric is green | Low to medium: the customer leaves the chat and calls the hotline, which adds load on support | Deferred to the backlog until a testable tone criterion exists |
| Wrong data in a source the bot sees: a broken tool or a wrong document (C-05, spread 1.5% instead of 0.9%) | Faithfulness compares the answer with what the bot itself retrieved, so a wrongly repeated value scores 1.0. Only the domain check catches it, because it takes correct facts from the rules engine | High: the customer is shown a wrong amount | Fix the tool (D20) now; keep the domain check in the blocking merge gate |

---

## 5. Run schedule

| Frequency | What goes in | Split criterion | Price |
| --- | --- | --- | --- |
| Every merge (blocks) | All 13 cases, domain check only | the check is deterministic and needs no judge, but the 13 agent calls still cost money; checks the mandate directly | about USD 0.014 per pass (agent calls only, section 7) |
| Nightly | Faithfulness and hallucination, all cases, 3 runs | LLM-judged, affordable overnight; hallucination separates clean from broken (0.0 vs 1.0) | about USD 0.05 per pass |
| Before release | Full set, all metrics, 3 runs plus 2 baseline runs, human review of red cases | answer relevancy is noisy and needs a human | about USD 0.05 per pass, paid once per release |

---

## 6. Localisation of one red case

Case: C-05 (CUS-0005, 6,000 EUR to USD) · Red metric: faithfulness 0.667 (lesson-02, run 1), domain check fails in all 3 runs

| Question | Answer |
| --- | --- |
| Layer | Generation |
| Specification line | `prompts/overlays/f5b472db63ad.md`, lines 10-13, replaces section "7. Output format": "the final amount you state to the customer must always be rounded UP to the nearest whole hundred ... Do NOT recompute it from the components ... do NOT show the unrounded value anywhere — a tidy bottom line matters more than arithmetic consistency." It contradicts `base.v1.md` line 60 ("a final amount consistent with them"). |
| Rewritten requirement | Now: the final amount must equal the quote tool result, shown with cents and consistent with the components shown. Never round. |
| — observable output | the final amount in the answer |
| — criterion | equals the `final_amount` field of the quote tool result, to the cent |
| — violation example | "Final amount: 6,500 USD (rounded up to the nearest hundred)" when the tool returned 6,423.91 |
| Hypothesis of the fix | Delete the rounding rule and use the requirement above. |
| Expected metric shift | Faithfulness for C-05: from about 0.67 to about 0.9 (the number is supported by the tool again). The domain check stays failed until the tool defect (spread 1.5% instead of 0.9%, D20) is fixed. |

Evidence: in 5 runs, lesson-02 gave 6,500.00 in 4 of 5 and 6,423.91 in 1 of 5; clean gave 6,463.04 in 5 of 5. The quote tool was called in 5 of 5 on both profiles and never returned 6,500. The hypothesis was written before any fix was applied.

---

## 7. Cost of a full run

| What | Value | Source |
| --- | --- | --- |
| Model calls per pass (13 cases) | 41 by the formula (cases x (1 + metrics)); about 124 actual (31 agent + 93 judge) | formula / cost block of the script report |
| Average call length, tokens | about 3,000 in / 310 out (agent calls) | script report |
| Price | USD 0.1 in / 0.5 out per million tokens (Haiku 5.5, prompts under 100k tokens) | script default, confirmed by the console Cost page (see below) |
| Cost of one pass | about USD 0.05 (agent 0.014 + judge 0.036) | measured from the report |
| Cost of the whole report (5 passes: 2 clean + 3 lesson-02) | about USD 0.25 | measured from the report |
| Provider console, 11 Oct 2026 (UTC), all of Haiku 5.5 that day | 5,748,129 tokens in, 2,233,059 tokens out, **USD 1.69** | console, Usage and Cost pages |

Formula: `run cost = calls x average tokens per call x price per token`

Substituted for one pass (13 cases), from the report (5 passes: 157 agent calls with 473,962 tokens in and 49,102 out, 465 judge calls):
- agent: 31.4 calls x (3,019 in x 0.1 + 313 out x 0.5) / 1,000,000 = 31.4 x USD 0.000458 = **USD 0.0144**
- judge: 93 calls x USD 0.000388 per call (`judge.cost_usd` of the report: USD 0.1806 over 465 calls) = **USD 0.0361**
- one pass: 0.0144 + 0.0361 = **USD 0.0505**, rounded to about USD 0.05; the merge gate (agent only) is the first line, USD 0.014

**Check against the console.** The price is confirmed: 5.748 M x 0.1 + 2.233 M x 0.5 = USD 1.69, exactly the total on the console Cost page. The console total for the day is about 7 times the cost of the final report (USD 0.25). That is plausible: the day also contains an earlier run, chat tests on the stand, and the five-run comparisons used for the localisation (section 6). The same API key and workspace are also used for studying with Claude in parallel, and the projects were not separated, so the console cannot show what this evaluation alone cost. The chart cannot separate one run from the rest either, so the console gives an upper bound for one run (USD 1.69) and confirms the price and the order of magnitude, not the exact USD 0.05. The per-run figure therefore comes from the script report, which counts only the calls the script made. A separate workspace or API key for the evaluation would make the console figure exact. The judge reasons before answering, so its real cost may be above our estimate; the next run should record it.

**Cost per month** (assumption: 10 merges a day, 22 working days, 2 releases a month):

| Run | Content | Per run | Count per month | Per month |
| --- | --- | --- | --- | --- |
| Every merge | 1 pass, domain check only (agent calls, no judge) | about USD 0.014 | 220 | about USD 3 |
| Nightly | 3 passes, faithfulness and hallucination | about USD 0.15 | 30 | about USD 4.5 |
| Before release | 5 passes, all metrics | about USD 0.25 | 2 | about USD 0.5 |
| **Total** | | | | **about USD 8 a month** |

Running the LLM metrics on the full set at every merge would cost about USD 0.05 x 220 = USD 11 a month for a single pass. Money is not the main argument against it: the same metrics stop 6 of 24 correct answers even at the lowest threshold (section 3), so every merge would produce false alarms. That is why the merge gate stays deterministic.
