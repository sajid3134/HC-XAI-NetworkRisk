# SecureSpeak — Complete Diagnostic Prescription

### A full technical and strategic diagnosis, prepared for review with a supervisor or consultant

**Prepared for:** Khondokar Sajid (and team: Abdul Alim Rakib, Sumaya Shimu Rima, Shakib Ahamed)
**Project:** SecureSpeak — CSE 498R Directed Research, North South University
**Supervisor:** Dr. Nova Ahmed
**Document purpose:** An honest, point-by-point diagnosis of the entire project — datasets, architecture, methodology, results, and every gap — so that a consultant or supervisor can quickly understand the true state of the work and advise on the fix.

> **How to read this document.** It moves from the top (what the project is) down to the machinery (exactly how each piece works), then to the problems (what is broken and why), then to the prescription (what to do). Sections are numbered so you can point a consultant directly to a section. Nothing here is inflated. Where a result is weak or wrong, it is labelled clearly. That is the point of this document.

---

## Table of Contents

1. Executive Summary — the one-page truth
2. What SecureSpeak Was Supposed To Be
3. The Datasets — what we used, what each is for, what is wrong with each
4. The System Architecture — every component, precisely
5. The ECAFN Fusion Engine — the centerpiece, explained in full
6. The Methodology — how everything was trained and evaluated
7. The Results — what is real, what is not
8. The Gaps — every problem, ranked by severity
9. Root-Cause Analysis — WHY each problem happened
10. What Is Genuinely Strong (and Publishable)
11. The Prescription — exactly what to do, in order
12. Questions To Ask The Consultant / Supervisor
13. Appendix A — File and Notebook Inventory
14. Appendix B — Glossary of Every Technical Term

---

# 1. Executive Summary — the one-page truth

SecureSpeak is a mobile security system that detects phishing and malware aimed at Bangladeshi Mobile Financial Services (MFS) users — bKash, Nagad, Rocket. It combines three detectors (URL phishing, Bangla SMS smishing, network anomaly) and fuses their outputs using a custom architecture called **ECAFN** (Evidence-Calibrated Attention Fusion Network) that uses "context" (which app, which port, what time, etc.) to decide how to combine the signals. It then shows the user a warning in Bangla and English.

**What is genuinely real and strong:**
- The **URL phishing detector** works: ~99.76% accuracy on real data.
- The **Bangla SMS smishing detector** works: ~98.2% accuracy. This is genuinely valuable because almost nobody builds Bangla-language security tools.
- The **bilingual, low-literacy warning system** is a real, novel, human-centered contribution.
- The **problem being solved is real and socially important**: vulnerable users losing money to phishing.

**What is broken or overstated (the honest core of this document):**
1. **The ECAFN fusion engine was trained on synthetic (computer-generated) data**, not real signals. Its labels come from a hand-written formula (`0.7·pp + 0.3·ap + noise`). This means it partly learned to reproduce a formula we wrote, not to detect real attacks.
2. **The evaluation set for the fusion was also synthetic** — signal values drawn from random distributions, not from real URLs and flows.
3. **The network anomaly detector cannot score the real phone-traffic data (PCAPdroid).** It was trained on CICFlowMeter's 80 features, but PCAPdroid records completely different columns. Every real flow scored identically. This means the headline "0% false positives on 52,118 real flows" is trivially true — the model could not actually "see" those flows.
4. **The central scientific claim — "our architecture beats simpler methods" — does not hold under honest testing.** When tested fairly (real non-linear labels, no hand-coded shortcuts), a plain neural network (MLP) matches or beats ECAFN on every test we ran (accuracy, zero-day generalization, low-data efficiency, false-positive suppression).
5. **ECAFN contains hand-coded shortcuts** (e.g., "if port is 4444, return HIGH") that bypass the learned model and would be flagged by any reviewer.

**The bottom-line diagnosis:**
The **project** is publishable, but **not as an "architecture" paper**. The fusion architecture is not a defensible headline contribution because it does not beat simple baselines. The real contributions are the working detectors (especially Bangla SMS), the bilingual warning design, the real-device deployment framing, and a planned user study. The correct move is to **reframe the paper around human-centered mobile security for vulnerable users**, describe the fusion honestly as a lightweight component, and target a usable-security / ICT4D venue (SOUPS, ACM COMPASS) rather than a systems/ML venue that expects architectural novelty.

---

# 2. What SecureSpeak Was Supposed To Be

### 2.1 The original goal

The project set out to protect Bangladeshi MFS users from three threats:
- **Phishing** — fake bKash/Nagad/Rocket login pages that steal PINs.
- **Smishing** — fraudulent SMS in Bangla/Banglish luring users to those pages.
- **Anomalies / malware, including zero-day** — malicious network behavior after a user taps a bad link, including malware families never seen before.

### 2.2 The intended intellectual contribution

The team's idea evolved like this (this is the honest "journey" of the project):
1. Start with three separate detectors (URL, SMS, network).
2. Notice that combining them with a **plain formula** (weighted average) is naive — it ignores the situation.
3. Introduce **context**: the same detector score means different things depending on which app made the request, what port it used, what time it was, and whether a money transaction was happening.
4. Build an **architecture (ECAFN)** that uses context to (a) decide how much to trust each detector and (b) resolve cases where detectors disagree.
5. Add **bilingual warnings** so non-technical users can act on the result.

This journey is genuinely interesting. The problem is that **step 4 — the architecture — was never proven to actually beat the plain formula or a simple neural network** once tested honestly. That is the core issue this document diagnoses.

### 2.3 Why the target population matters

The intended users are elderly pensioners, first-time smartphone owners, rural remittance recipients, and daily-wage earners. They often cannot read English security alerts and cannot recognize a phishing URL. This makes the **human-centered angle** (Bangla warnings, low-literacy design) the most defensible and socially valuable part of the work — a point we return to in Section 10.

---

# 3. The Datasets — what we used, what each is for, what is wrong with each

This section is critical for the consultant. Several problems trace directly back to dataset choices.

### 3.1 StealthPhisher 2025 (URL phishing)

- **What it is:** ~150,000 URLs labeled phishing or legitimate. Created at GLA University, Mathura, India, published early 2025. Sourced partly from PhishTank.
- **Role:** Training the URL phishing detector.
- **Status:** **REAL and usable.** This is one of the solid parts.
- **Known concern (raised by a teammate):** possible bias where benign URLs contain "www." and phishing ones do not, letting a model "cheat." **Our check:** the model's top features (via SHAP) are structural (slash count, path length, URL length, entropy), and no `has_www` feature was engineered, so the model is probably not exploiting that shortcut — but this was never fully audited by stripping "www." and re-measuring. **Action:** run that audit (see Section 11).

### 3.2 BangalaBarta 2024 (Bangla SMS smishing)

- **What it is:** 4,374 Bangla SMS messages labeled smishing or normal.
- **Role:** Training the Bangla SMS detector.
- **Status:** **REAL and valuable.** This is arguably the single most novel dataset in the project because Bangla-language security resources barely exist.
- **Concern:** it is small (4,374 messages) and not exclusively MFS-focused, but it is defensible.

### 3.3 CICMalAnal2017 (network malware flows)

- **What it is:** ~2,126 CSV files of Android network flows across malware families (Adware, Ransomware, SMS-malware, Scareware) and benign, in **CICFlowMeter 80-feature format** (Flow Duration, Total Fwd Packets, Flow Bytes/s, etc.).
- **Role:** Training the network classifier + anomaly detector; SCAREWARE was held out to test "zero-day" detection.
- **Status:** **REAL, but see the critical mismatch below.**

### 3.4 UNSW-NB15 (secondary network dataset)

- **What it is:** A different network intrusion dataset (45 columns: dur, proto, spkts, dbytes, etc.).
- **Role:** Appears in the notebook as an alternate/again network source.
- **Status:** REAL, but its presence adds confusion — the paper says CICMalAnal2017 while the code also references UNSW-NB15. **This inconsistency must be resolved** (Section 8).

### 3.5 PCAPdroid Bangladesh (real device traffic) — **THE CRITICAL PROBLEM**

- **What it is:** 52,118 real network flows captured from 6 volunteer Bangladeshi Android phones during normal use (bKash, WhatsApp, Facebook, YouTube). Columns: `BytesSent, BytesRcvd, PktsSent, PktsRcvd, DstPort, IPProto, FirstSeen, LastSeen, App, PackageName`, etc.
- **Intended role:** Prove the system does not raise false alarms on real Bangladeshi traffic ("0% false-positive rate on 52,118 real flows").
- **Status:** **REAL data, but UNUSABLE by the current network model — this is a central defect.**
- **Why:** The network model was trained on **CICFlowMeter's 80 features**. PCAPdroid records **completely different columns**. When the code tried to map PCAPdroid into the model's feature space, **75+ of the 80 features became zero**, so **every one of the 52,118 flows scored identically** (all collapsed to a single anomaly value of 0.0819). 
- **Consequence:** The "0% false positives on 52,118 real flows" claim is **trivially true and misleading** — the model could not actually distinguish the flows; it gave them all the same score, and that score happened to be below the alarm threshold. A reviewer who checks this will reject the claim immediately.
- **This is not the user's fault in spirit** — the data is real and was captured properly — but the **feature spaces are fundamentally incompatible**, and the raw `.pcap` files (which could have been reprocessed through CICFlowMeter) were not saved; only the CSV survived.

---

# 4. The System Architecture — every component, precisely

This section describes exactly how the system is built, component by component. Terms are defined in Appendix B.

### 4.1 High-level flow

```
   URL  ─────►  URL Detector (Random Forest)  ────► pp  (phishing probability 0–1)
   SMS  ─────►  SMS Detector (MiniLM + LogReg) ────► sms score
   Flow ─────►  Network Detector (XGBoost +          
                FlowAE autoencoder + IsolationForest) ► ap  (anomaly probability 0–1)

   Device context (app, port, time, transaction state ...) ► 12-D context vector

   pp, ap, context ─────► ECAFN fusion engine ─────► final risk (HIGH / MEDIUM / LOW)
                                                       │
                                                       ▼
                                          Bilingual warning (Bangla + English)
```

### 4.2 URL phishing detector (Step 1A in the notebook)

- **Input:** a URL string.
- **Features:** 26 hand-engineered numbers per URL, in four groups:
  - Structural: URL length, dot count, slash count, digit ratio, letter ratio.
  - Lexical: domain length, subdomain depth, TLD risk.
  - Bangladesh-specific: bKash/Nagad/Rocket keyword presence, brand-impersonation score.
  - Suspicion: URL entropy, free-hosting TLDs, financial keywords, presence of `@`, IP-address host.
- **Model:** several classifiers were compared (RandomForest, XGBoost, LightGBM, LogisticRegression, SVM); the best (RandomForest, 300 trees, class-balanced) was chosen.
- **Validation:** 5-fold cross-validation with the scaler refit inside each fold (correct — no leakage).
- **Result:** ~99.76% accuracy, ~99.93% AUC, 5-fold F1 ~99.64% ± 0.09%.
- **Verdict:** **REAL and solid.**

### 4.3 Bangla SMS smishing detector (Step 1B)

- **Input:** an SMS string (Bangla, Banglish, or English).
- **Features:** a 384-dimensional sentence embedding from `paraphrase-multilingual-MiniLM-L12-v2` (a small multilingual language model), concatenated with 6 hand-crafted features (URL present?, MFS keywords?, urgency words?, length ratio, exclamation density, digit ratio) → 390 numbers total.
- **Model:** class-weighted Logistic Regression.
- **Why MiniLM and not BanglaBERT:** MiniLM is ~7.5× smaller and ~47× faster, which matters for cheap phones. Reasonable, defensible choice.
- **Result:** ~98.2% accuracy, ~99.82% AUC.
- **Verdict:** **REAL and valuable** (the most novel detector).

### 4.4 Network anomaly detector (Steps 2–3)

Three sub-components working together:
- **XGBoost classifier** — trained on 80 CICFlowMeter features to classify known malware families.
- **FlowAE** — a symmetric autoencoder (Linear → BatchNorm → GELU → Dropout, 128-dim latent) that learns to reconstruct benign flows; a high reconstruction error signals a possible unseen (zero-day) attack.
- **IsolationForest** — an outlier detector that produces the normalized anomaly probability `ap` (0–1) via the saved `RMIN/RMAX` range constants.
- **Zero-day design:** the SCAREWARE family was physically held out of training, so testing on it is a genuine "never-seen-before" test.
- **Verdict on the models themselves:** REAL and reasonably built. **BUT** — see Section 3.5 — this detector **cannot score the PCAPdroid real-device data**, which undermines the real-world evaluation.

### 4.5 Bilingual warning generator (Step 5)

- **Primary path:** Groq LLaMA-3.1 (8B) generates a context-specific warning.
- **Fallback path:** rule-based templates in Bangla and English for every risk level, working fully offline.
- **Example (HIGH risk, Bangla):** "⚠️ সতর্কতা: এই লিংকটি ফিশিং হতে পারে। আপনার PIN দিবেন না। bKash অ্যাপে ফিরে যান।"
- **Verdict:** **REAL, novel, and the strongest human-centered artifact in the project.**

---

# 5. The ECAFN Fusion Engine — the centerpiece, explained in full

This is the most important section, because ECAFN was meant to be the paper's core contribution, and it is where the deepest problems live.

### 5.1 What ECAFN is supposed to do

Combine `pp` (URL phishing probability) and `ap` (network anomaly probability) into a single risk score, using a 12-dimensional **context vector** to decide how to combine them. The intuition: the same `pp=0.5, ap=0.5` should mean "safe" if it comes from the real bKash app on a normal port at 2pm, but "attack" if it comes from an unknown app on port 4444 at 3am.

### 5.2 The 12-dimensional context vector

Computed by rules (not learned) from device metadata:

| # | Dimension | Meaning |
|---|-----------|---------|
| 1 | app_trust | Is the app a known-good MFS/social app (0) or unknown/suspicious (up to 1)? |
| 2 | port_trust | Is the destination port a safe one (443, 80…) or not? |
| 3–4 | time_sin, time_cos | Time of day, encoded cyclically. |
| 5–7 | bytes, packets, duration | Flow size/rate features (log-scaled). |
| 8 | proto_risk | Protocol risk (HTTPS low, others higher). |
| 9 | mfs_context | Is a money transaction in progress? |
| 10 | brand_impersonate | Does an unverified app mimic an MFS brand name? |
| 11 | pp·ap | Interaction of the two signals. |
| 12 | \|pp − ap\| | How much the two signals disagree. |

**This part is legitimate** — these are deterministic, explainable rules computed from real metadata fields.

### 5.3 The ECAFN internal pipeline (four stages)

1. **ReliabilityMLP** (1,946 parameters): takes the 12-D context → outputs two "trust weights" `r_pp, r_ap`. Calibrated signals: `pp_c = pp · r_pp`, `ap_c = ap · r_ap`. (This is context entering *before* fusion.)
2. **Dempster–Shafer combination:** combines `pp_c, ap_c` into a belief `bt` and an uncertainty `unc` using evidence-theory math.
3. **Cross-Attention Gated Fusion (CGF)** (561 parameters): uses the context as an attention "query" over the calibrated signals to produce `caf`. (This is context entering *during* fusion.)
4. **Smooth conflict gate:** blends the DST belief and the CGF output based on how much the signals agree: `w = 1 − σ(5·|pp_c − ap_c|)`, `final = w·bt + (1−w)·caf`.

**The architecture itself is coherent and reasonable.** The problem is not the design — it is how it was trained and whether it actually helps. That is next.

### 5.4 PROBLEM 1 — ECAFN was trained on synthetic data with a near-linear label

In the training cell, the fusion's training targets were generated like this (paraphrased from the code):

```
score  = 0.7 · pp_calibrated + 0.3 · ap_calibrated + noise
label  = 1 if score > 0.45 else 0
```

- The `pp` and `ap` values themselves were **drawn from random distributions** (`np.random.beta`, `np.random.normal`), **not from real URLs or flows**.
- The label is essentially a **linear formula** of `pp` and `ap`.

**Two consequences:**
- (a) ECAFN partly learned to **reproduce a formula the team wrote**, which is not the same as learning to detect real attacks.
- (b) Because the target is basically linear, **any linear or simple model can match it** — which is exactly why ECAFN never convincingly beats a plain neural network.

### 5.5 PROBLEM 2 — The fusion evaluation set was also synthetic

The "conflict set" used to report ECAFN's headline numbers was built with a random generator (`rng_ev`), sampling `pp` and `ap` from uniform ranges and assigning labels by rule — **not from real URLs paired with real flows.** So the reported fusion numbers describe performance on invented data, not real data.

### 5.6 PROBLEM 3 — Hand-coded shortcuts inside ECAFN

The ECAFN prediction function contains rules that bypass the learned model, for example (paraphrased):

```
if use_port_override and port in MALICIOUS_PORTS:
    return HIGH, 0.90            # skip the learned fusion entirely
if ap > 0.45 and pp < 0.15:
    boost the score
```

**Why this is a problem:** these rules do some of the detection work by hand, so any "win" partly comes from the rules, not the architecture. A reviewer will call this "hard-coding the answer." When these overrides are turned off (as they must be for an honest test), ECAFN's performance is just that of the learned fusion — which does not beat a plain MLP.

### 5.7 The honest architecture test (what we ran to verify)

We re-implemented ECAFN cleanly (no shortcuts) and tested it against a plain MLP and a linear model, using labels that genuinely depend on context in a **non-linear** way (so the task actually rewards good context modeling). Four tests:

| Test | Question | Result |
|------|----------|--------|
| Accuracy on hard cases | Does ECAFN beat MLP when both signals are weak? | **No — tie / MLP slightly ahead** |
| Zero-day generalization | Does ECAFN degrade less on unseen attack types? | **No — MLP degraded less (drop 0.002 vs 0.012)** |
| Low-data efficiency | Does ECAFN need fewer training samples? | **No — MLP higher AUC at every size, even N=40** |
| False-positive suppression | Does ECAFN raise fewer false alarms on trusted-context traffic? | **Tie — all methods 0% on this easy check** |

**Conclusion of the test:** across every setting designed to give the architecture its best chance, **a plain MLP matches or beats ECAFN.** This is expected in machine learning: on small tabular problems, once the context features are provided as inputs, a simple model learns any needed interactions, and structured architectures rarely add value. (Structured architectures shine on images, text, and graphs — not 14-column tabular fusion.)

---

# 6. The Methodology — how everything was trained and evaluated

### 6.1 The training pipeline (the "master notebook")

One notebook trains everything in sequence under a fixed random seed (42):
1. Train URL detector on StealthPhisher.
2. Train SMS detector on BangalaBarta.
3. Load CICMalAnal2017, hold out SCAREWARE, train XGBoost + FlowAE + IsolationForest.
4. Train ECAFN fusion (on synthetic data — Problem 1).
5. Set up bilingual warnings.
6. Real-world evaluation on PCAPdroid (broken by the feature mismatch — Section 3.5).
7. Failure analysis.
8. Save 16 model artifacts + a manifest.

### 6.2 The evaluation notebooks (SS_0 – SS_3)

- **SS_0:** confirms URL/SMS metrics + zero-day + the (broken) PCAPdroid FPR.
- **SS_1:** the BORDERLINE headline (87.11%) — but on a synthetic conflict set.
- **SS_2:** SHAP interpretability + URL-only baseline.
- **SS_3:** end-to-end injection — but partly synthetic context.

### 6.3 The methodological problems, summarized

1. **Circular / synthetic evaluation:** the fusion is trained and tested on data generated by formulas, so the headline fusion numbers do not describe real-world performance.
2. **Feature-space mismatch:** the network model cannot score the real PCAPdroid data, so the real-device claims are hollow.
3. **Baselines were weak or hand-tuned:** the DST-only baseline was admitted to be under-tuned (belief-only thresholding gave 12.37%), which unfairly flattered ECAFN.
4. **No external SOTA baselines:** no comparison to URLTran, BanglaBERT, or a fair context-as-features MLP was completed.
5. **No user study yet:** the warning system — the strongest artifact — has no human evaluation.

---

# 7. The Results — what is real, what is not

| Result | Reported | Honest status |
|--------|----------|---------------|
| URL detection accuracy | 99.76% | **REAL** (on real StealthPhisher data) |
| URL 5-fold CV F1 | 99.64% ± 0.09% | **REAL** |
| SMS detection accuracy | 98.20% | **REAL** (on real BangalaBarta data) |
| Zero-day SCAREWARE detection | (varies) | **PARTIALLY REAL** — real held-out family, but low detection; honest but weak |
| BORDERLINE detection 87.11% | headline | **NOT REAL** — measured on synthetic conflict set |
| 0% FPR on 52,118 real flows | headline | **MISLEADING** — model could not actually score those flows (all identical) |
| +16.5 pp over URL-only | headline | **NOT RELIABLE** — from synthetic/partly-synthetic evaluation |
| DST-only ablation 12.37% | ablation | **UNFAIR** — baseline was under-tuned |
| ECAFN beats MLP | implied | **FALSE under honest testing** — MLP matches/beats ECAFN |

**The two numbers you can stand behind in front of anyone:** URL 99.76% and SMS 98.20%, both on real data. Almost everything about the *fusion* needs to be re-described honestly.

---

# 8. The Gaps — every problem, ranked by severity

**CRITICAL (must fix or drop before any submission):**
1. Fusion trained on synthetic data with a near-linear label.
2. Fusion evaluated on synthetic data.
3. Network model cannot score real PCAPdroid data (feature mismatch); "0% FPR on 52k flows" is hollow.
4. Architecture does not beat a plain MLP under honest testing.
5. Hand-coded shortcuts inside ECAFN.

**MAJOR (will draw reviewer objections):**
6. No external SOTA baselines (URLTran, BanglaBERT, context-as-features MLP).
7. DST baseline under-tuned (unfair ablation).
8. Dataset inconsistency: paper says CICMalAnal2017, code also uses UNSW-NB15.
9. Small hard-case set (194 borderline URLs).
10. No user study for the warning layer.

**MINOR (polish):**
11. "www." bias in StealthPhisher not fully audited.
12. Raw `.pcap` files not saved (only CSV), limiting reprocessing options.

---

# 9. Root-Cause Analysis — WHY each problem happened

- **Why synthetic fusion data?** Pairing *real* borderline URLs with *real* matching flows and *real* context is genuinely hard — you need aligned (URL, flow, context, label) tuples. Generating them synthetically was a shortcut to get the pipeline running. Understandable, but it undermines the scientific claim.
- **Why the PCAPdroid mismatch?** The network model was trained on CICFlowMeter features (a research standard), but real phones capture PCAPdroid-style connection logs, which lack the per-packet statistics CICFlowMeter computes. These two formats are fundamentally different, and the raw packet captures that could bridge them were not saved.
- **Why does the architecture not beat an MLP?** Because the context features are already provided as inputs; a plain MLP can learn any interaction among them. On small tabular data, structured architectures rarely beat good simple models. This is a known, expected ML result — not a coding bug.
- **Why the shortcuts?** They were added to make the system "work" on obvious cases (a known-malicious port should obviously be flagged), but they contaminate the evaluation of the learned model.

---

# 10. What Is Genuinely Strong (and Publishable)

Even after all the above, there is a real paper here — just not the one originally imagined. The honest, defensible contributions:

1. **Bangla/Banglish smishing detection (98.2%).** Genuinely under-served; very few security tools handle Bangla. This alone has value.
2. **A complete, working end-to-end mobile security system** for a real, high-stakes, under-served population (Bangladeshi MFS users).
3. **Bilingual, low-literacy security warnings** — a novel human-centered artifact grounded in the needs of elderly and non-technical users.
4. **Real-device deployment framing + an honest threat model** (clearly stating what is and is not covered — e.g., it does not stop "authorized push payment" fraud where a user willingly sends money).
5. **A planned 35-person user study** — the kind of real human evidence that usable-security venues value and most technical papers lack.

**The right framing:** a **human-centered / usable-security / ICT4D** paper about protecting vulnerable users, with the fusion described honestly as "a lightweight context-aware fusion layer" — not oversold as a novel architecture that beats baselines.

**The right venues:** SOUPS (Symposium on Usable Privacy and Security) or ACM COMPASS (Computing and Sustainable Societies), where Dr. Nova Ahmed has a track record and where exactly this kind of work belongs. Not a top ML/systems venue that expects architectural novelty.

---

# 11. The Prescription — exactly what to do, in order

### Phase A — Stop the bleeding (decide what to keep, drop, and fix)
- **KEEP:** URL detector, SMS detector, bilingual warnings, the threat model, the problem framing.
- **DROP or DEMOTE:** the claim that ECAFN beats baselines; the synthetic fusion numbers; the "0% FPR on 52k flows" claim in its current form.
- **FIX honestly:** re-describe the fusion as a working component, not a proven-superior architecture.

### Phase B — Make the network side honest (choose one)
- **Option 1 (best if possible):** if any raw `.pcap` files can still be recovered, reprocess them through the real CICFlowMeter tool so the existing model can score them properly.
- **Option 2 (practical, recommended):** rebuild the network anomaly detector on the ~11 features that BOTH CICMalAnal2017 and PCAPdroid actually share (bytes, packets, duration, port, protocol, ratios). This makes the detector work on real phone-capturable data — which is actually a stronger deployment story. (A notebook for this was drafted; it needs the "hard set" issue resolved so the evaluation is non-trivial.)

### Phase C — Strengthen the honest contributions
- **Run the 35-person user study** on the bilingual warnings (comprehension, trust, workload/NASA-TLX, intended action, think-aloud). This is the single highest-value next step and can be done in ~1 week.
- **Add fair baselines** for the detectors you keep: URLTran (URL), BanglaBERT (SMS), and — if you keep any fusion claim — a context-as-features MLP, reported honestly even if it ties.
- **Audit the "www." bias** by stripping "www." and re-measuring URL accuracy.
- **Resolve the dataset inconsistency** (decide CICMalAnal2017 vs UNSW-NB15; state clearly).

### Phase D — Reframe and write the paper
- Reframe around human-centered mobile security for vulnerable users.
- Present the fusion honestly as a lightweight component.
- Lead with: Bangla smishing + bilingual warnings + user study + real-device deployment + honest threat model.
- Target SOUPS or ACM COMPASS.

### Phase E — Optional future work (only if a consultant sees a path)
- If a consultant identifies a genuinely novel, testable property of the fusion (something a plain MLP cannot do), pursue it — but only if it can be shown honestly on real data. Do not invest more time forcing an architecture-superiority claim the data does not support.

---

# 12. Questions To Ask The Consultant / Supervisor

Bring these exact questions:
1. Given that the fusion architecture does not beat a plain MLP on honest tests, is reframing to a **human-centered usable-security paper** the right move — or is there a fusion property worth testing that we missed?
2. For the network side: is it acceptable to **rebuild the anomaly detector on PCAPdroid-native features** (Option 2), and present it as a deployability strength?
3. Is a **35-person user study** sufficient for the venue we target (SOUPS / COMPASS), and what study design would you want to see?
4. Which of Dr. Nova Ahmed's own venues/papers should we align with and cite?
5. Is the **honest threat model** (covering credential phishing/smishing/malware but NOT authorized-payment fraud) a strength to foreground?
6. Given the timeline (goal: visible result before Oct/Nov), should we target a **preprint + conference** now and a journal later?

---

# 13. Appendix A — File and Notebook Inventory

**Saved model artifacts (in `SecureSpeak_Output/saved_models/`):**
- `url_model.joblib`, `url_scaler.joblib`, `url_meta.json` — URL detector (REAL, good)
- `sms_model.joblib`, `sms_scaler.joblib`, `sms_meta.json` — SMS detector (REAL, good)
- `net_model.joblib`, `net_scaler.joblib`, `net_meta.json` — network classifier (REAL, but cannot score PCAPdroid)
- `flowae.pt`, `iso_forest.joblib`, `ap_norm.json` — anomaly components
- `reli_mlp.pt`, `cgf.pt`, `thresholds.json` — ECAFN fusion (trained on synthetic data)
- `model_manifest.json` — verification stamp

**Newer artifacts from the honest-fix attempts (in `SecureSpeak_Output/ecafn/`):**
- `net_scaler_v2.joblib`, `iso_forest_v2.joblib`, `ap_norm_v2.json` — rebuilt anomaly detector on shared features (REAL, works on PCAPdroid)
- `borderline_eval_set.parquet` — evaluation set (went through several rebuilds; see notes)

**Notebooks:**
- `SecureSpeak_Blackbook_version.ipynb` — the master training notebook (54 cells)
- `Q1_HONEST_FIX_all_in_one.ipynb` — retrains fusion on real eval set
- `Q1_REBUILD_network_detector.ipynb` — rebuilds anomaly detector on shared features
- `Q1_BUILD_hard_conflict_set.ipynb` — attempts a genuinely hard evaluation
- `Q1_ARCHITECTURE_TEST.ipynb` — the honest ECAFN-vs-MLP test

**Datasets (in `Datasets/`):**
- `StealthPhisher2025.csv` (URL, real, good)
- `BangalaBarta ... .csv` (Bangla SMS, real, valuable)
- `CICMalAnal2017/` (2,126 CSVs, CICFlowMeter format)
- `UNSW_NB15_training-set.csv` (secondary, causes inconsistency)
- `pcapdroid_merge.csv` (52,118 real flows, incompatible with CICFlowMeter model)

---

# 14. Appendix B — Glossary of Every Technical Term

- **pp** — "phishing probability," the URL detector's output (0–1).
- **ap** — "anomaly probability," the network detector's output (0–1).
- **Context vector** — 12 numbers describing the situation (app, port, time, transaction state, etc.).
- **ECAFN** — the custom fusion engine combining pp, ap, and context.
- **ReliabilityMLP** — small network that turns context into trust weights for each detector.
- **Dempster–Shafer (DST)** — a mathematical theory for combining uncertain evidence into a belief + uncertainty.
- **CGF (Cross-Attention Gated Fusion)** — uses context as an attention query over the signals.
- **MLP (Multi-Layer Perceptron)** — a plain neural network; the honest baseline the architecture must beat (and does not).
- **BORDERLINE** — URLs whose phishing probability is ambiguous (pp between 0.30 and 0.70).
- **Zero-day** — an attack type never seen during training (here, the held-out SCAREWARE family).
- **FlowAE** — an autoencoder that flags unusual flows by how badly it reconstructs them.
- **IsolationForest** — an algorithm that scores how "outlier-like" a flow is.
- **CICFlowMeter** — a tool that computes 80 detailed flow features from packet captures.
- **PCAPdroid** — an Android app that captures network connection logs (fewer, different features than CICFlowMeter).
- **SHAP** — a method to explain which features drive a model's decisions.
- **AUC** — Area Under the ROC Curve; a threshold-independent accuracy measure (0.5 = random, 1.0 = perfect).
- **FPR** — False Positive Rate; fraction of benign cases wrongly flagged.
- **SOUPS / COMPASS** — usable-security and computing-for-society venues that fit this work.

---

*End of diagnostic prescription. This document reflects the honest state of SecureSpeak as understood from the notebooks and data. The purpose is not to discourage — the project solves a real problem and contains genuinely valuable, publishable work — but to give a consultant or supervisor a precise, unvarnished map so the fix can be fast and correct.*
