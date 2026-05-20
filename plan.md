The SecureSpeak 12-Week Upgrade Plan: First to Last
Before Anything — Two Ground Rules
Rule 1: One item at a time. Don't try to parallel-process. Finish item N completely before touching item N+1. This is how you avoid the mess you just felt.
Rule 2: Every change must be tested before the next change. After each item, you should be able to re-run the full pipeline and confirm it still works. If something breaks, we fix it before moving on.

PHASE 1: Foundation Audit (Week 1)
Goal: Know exactly what data and code you have, in detail, before changing anything. No new training, no new datasets, no new experiments — just understanding.
Item 1.1 — Dataset health check on what you already have
Run a structured audit on each of your current datasets. For each: row count, column names, dtypes, NaN counts, infinite values, duplicates, class balance, label encoding correctness.
Datasets to audit:

StealthPhisher 2025 (URL classifier source) — critical, verify the "Phishing: 0" issue I flagged
BangalaBarta 2024 (SMS classifier source) — verify the smish/promo merge
CICMalAnal2017 (network classifier source) — verify SCAREWARE separation is actually enforced
pcapdroid_merge.csv (your 52,118 real flows) — verify column structure matches CICMalAnal2017

Output: One short report (~1 page per dataset) with verdict: keep / fix / drop.
Item 1.2 — Re-run your existing v5 notebook end-to-end and confirm all numbers match the paper
This is the boring but essential step. You need to be able to reproduce 99.76% URL, 98.20% SMS, 79.84% network, 0.00% PCAPdroid FP from a clean run. If anything differs from the paper, we know the paper has stale numbers and we fix them before doing anything else.
Item 1.3 — Lock the codebase
Create a Git repository (GitHub, private for now). Commit the v5 notebook as the baseline. From here, every change is a commit. This means if something breaks later, you can roll back to a known-good state. Non-negotiable for Q1 submission — reviewers may ask for the repo.

PHASE 2: Kill the Synthetic-Evaluation Problem (Weeks 2–3)
Goal: Replace the synthetic 500-attack benchmark with a real-attack benchmark. This is the single biggest credibility upgrade.
Item 2.1 — Add PhishTank as external URL validation
PhishTank gives you fresh real phishing URLs your model has never seen. Download the live feed, deduplicate, run your existing URL classifier on it, report honest accuracy. Expect the number to drop from 99.76% to somewhere between 80–92% — that's normal and expected. The honest drop is what reviewers respect.
Item 2.2 — Add PhiUSIIL as second external URL validation
Same idea, different dataset (2024 URLs from UCI Repository). Provides cross-validation across two independent external sources. If your classifier holds up on both, that's a strong publishable result.
Item 2.3 — Build the real (pp, ap, context) test set
Instead of generating synthetic (pp, ap) scores in three tiers, build a real evaluation set:

Take 200 real PhishTank phishing URLs → run through your URL classifier → real pp scores
Take 200 real SCAREWARE flows (your zero-day holdout) → run through your network classifier → real ap scores
Pair them with realistic context vectors (use actual ports/protocols from the malware flows)
200 legitimate URL + benign flow pairs from PCAPdroid for the negative class

Test ECAFN, CATF, DST-only, and Attn-Fusion on this real set. Report honest numbers. This single replacement removes the synthetic critique permanently.
Item 2.4 — Re-tune ECAFN thresholds on real data
Your current thresholds were grid-searched on synthetic data. Re-grid-search them on a held-out portion of the real evaluation set. Update all results tables.

PHASE 3: Add Real Technical Novelty (Weeks 4–5)
Goal: Make at least one component genuinely novel, not just a recombination. Pick ONE of these — not all three.
Item 3.1 — Conformal Prediction for Calibrated Uncertainty (RECOMMENDED)
Replace your hand-tuned uncertainty parameters (u_p=0.08, u_n=0.15) with conformal prediction sets. This gives you mathematically guaranteed coverage probability — a sentence reviewers love. About 200 lines of Python on top of what you have. Library: mapie or crepes.
Why this one: It's cleanly publishable, well-understood theoretically, and gives you a one-line claim that reads like a Q1 paper: "SecureSpeak provides distribution-free finite-sample uncertainty quantification with user-specified coverage guarantees."
Item 3.2 — Replace synthetic CGF training with self-supervised pretraining
Pretrain the context encoder using contrastive learning on your real 52,118 PCAPdroid flows. Positives: flows from the same app within a short time window. Negatives: flows from different apps. Then fine-tune CGF on a smaller real-labeled set. This addresses the "trained on synthetic" critique at the architectural level.
Item 3.3 — Adversarial robustness evaluation
Add evasion-attack evaluation: small perturbations to URL features, network features, and context that try to flip your classification. Use PGD-style attacks adapted to your features. Show ECAFN is more robust than baselines under attack. Hot topic at 2024–2026 security venues.
My recommendation for you specifically: Item 3.1 (Conformal Prediction). It's the most tractable, most defensible, and most Q1-flavoured. Other two are more research-risky.

PHASE 4: Replace the Port-Override Heuristic (Week 6)
Goal: Remove the hand-coded if port in {4444, 8888, ...} rule and replace it with a learned port-risk embedding.
Item 4.1 — Build the port-risk embedding
Encode port as a learned 16-dimensional embedding. Train it jointly with the reliability calibration using port-to-malicious-frequency statistics aggregated from CICMalAnal2017 + your real PCAPdroid data.
Item 4.2 — Re-run all evaluations without the port-override branch
Report pure ECAFN performance. If the BORDERLINE detection rate drops from 99.5% to, say, 87%, that's honest and acceptable. The paper sentence becomes: "ECAFN operates without hand-tuned port rules." Reviewer trust shifts permanently in your favour.
Item 4.3 — Add SOTA baseline comparisons
At least two external published baselines, run on your real evaluation set:

For URL: URLTran (2021) or PhishLLM (2024)
For SMS: BanglaBERT fine-tuned on BangalaBarta
For network: any 2023–2024 published Android malware detector

These don't all need to beat ECAFN — if some do, you discuss the tradeoff (interpretability, on-device cost). What matters is showing reviewers you've engaged with the state of the art.

PHASE 5: The Real User Study (Weeks 5–9, overlaps with technical work)
Goal: Run the 35-user study properly so it lands at SOUPS or CHI. Start the protocol design early because IRB takes time.
Item 5.1 — IRB / Ethics approval from NSU
Submit ethics application immediately, week 5. This can take 2–4 weeks. Without IRB, no top-tier HCI venue will accept the paper. Document everything.
Item 5.2 — Pre-register hypotheses on OSF.io
Free, takes one hour. Pre-registration is a +1 with serious HCI reviewers.
Item 5.3 — Recruit 35 participants with proper stratification

10 elderly (60+)
10 low-literacy or limited-English
5 MFS agents
5 young digital natives
5 women specifically (gender gap in fintech access in BD is documented and worth addressing)
At least 3 from outside Dhaka

Item 5.4 — Three-condition within-subjects study design
Each participant sees three warning conditions in randomized order:

English-only generic warning (control)
Translated Bangla warning (translation API)
SecureSpeak native bilingual warning (your system)

Item 5.5 — Four measured outcomes

Comprehension (did they understand the threat?)
Intended action (close link / enter PIN / call helpline)
Confidence (Likert scale)
Trust calibration (do they correctly trust true alerts and dismiss false ones?)

Item 5.6 — Qualitative analysis
Record semi-structured interviews. Thematic analysis with a second coder (recruit one teammate). Report inter-rater agreement (Cohen's kappa).
Item 5.7 — Report failure cases
Users who DIDN'T understand even your Bangla warning. Reviewers trust papers that show their system's limits. This is a strength, not a weakness.

PHASE 6: Reproducibility, Ablations, Polish (Week 10)
Goal: Make the work publication-ready by 2026 standards.
Item 6.1 — Full ablation table
ECAFN with each component removed (ReliabilityMLP off, DST off, CGF off, port-embedding off). Confidence intervals on each cell. McNemar's tests for each pairwise comparison.
Item 6.2 — Computational cost analysis
Inference time on a mid-range Android device. Memory footprint. Estimated battery impact. Q1 papers in mobile security expect this.
Item 6.3 — Public reproducibility package

Anonymized PCAPdroid dataset (with IRB approval, with participant consent)
Code repository with frozen requirements, model checkpoints
Expected outputs hash-verified so reviewers can confirm reproduction
README with one-command reproduction script

Item 6.4 — Drop the weak claims

Remove "100% on conflict subset" framing (replace with honest real-data numbers from Phase 2)
Remove UNSW-NB15 8/22 features result entirely
Drop "first" claims, replace with specific verifiable scope claims
Drop the 3-tier OBVIOUS/BORDERLINE/STEALTH taxonomy as a "novel methodological contribution" — keep it as an evaluation protocol you adopted


PHASE 7: Write Both Papers (Weeks 10–12)
Goal: Two distinct papers from the same project.
Item 7.1 — Security paper (target: IEEE TDSC or Computers & Security)
Focus: ECAFN architecture, conformal uncertainty, real-data evaluation, port embedding, SOTA baselines. 12–15 pages. Technical depth. You lead this paper as first author.
Item 7.2 — HCI paper (target: SOUPS 2026 or CHI 2027)
Focus: bilingual warning design, the 35-user study, qualitative findings on elderly Bangladeshi users, the case for native-language security communication in the Global South. 10–14 pages. Human-centred depth. Dr. Nova Ahmed will likely be a strong co-author here given her research profile.
Item 7.3 — Workshop paper as insurance (target: any week, anytime)
Submit an early version to a workshop — CHI Late-Breaking Work, USENIX SOUPS Posters, or NDSS DLSP. Even if both main papers face delays, you have one publication landed before your Australia application cycle.What to Drop Entirely
These items in the current paper hurt you. Remove them:

The UNSW-NB15 cross-domain result (8/22 features is not interpretable)
The "100% accuracy on conflict subset" headline number (replace with honest real-data result)
The framing of 4.81% SCAREWARE zero-day as "motivating fusion" (move to honest limitations section)
"First mobile security framework for Bangladesh" claims (replace with specific scoped claims)
The 3-tier OBVIOUS/BORDERLINE/STEALTH framing as novel (reframe as adopted evaluation protocol)


What's the Minimum Viable Path?
If 12 weeks turns out to be too tight, here is the absolute minimum to publish something credible:

Phase 1 (audit) — non-negotiable
Phase 2 (real-data evaluation) — non-negotiable
Item 3.1 (conformal prediction) — non-negotiable for novelty
Phase 4 (port embedding + SOTA baselines) — non-negotiable
Phase 5 SHORT VERSION: 15 users instead of 35, single condition, comprehension test only
Phase 6 + 7 — non-negotiable

With this minimum, you can publish at IEEE TDSC or Computers & Security (security track, Q1) and at SOUPS Posters (HCI workshop track). Not two top-tier papers, but two real publications, one of which is Q1.
