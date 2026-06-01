SecureSpeak Research Package
============================
Team: Khondokar Sajid, Abdul Alim Rakib, Sumaya Shimu Rima, Shakib Ahamed
Supervisor: Dr. Nova Ahmed | NSU CSE 498R | Spring 2026

CONTENTS
--------

01_notebooks/        — All Jupyter notebooks in correct execution order
02_results/          — JSON results files from each notebook
03_documentation/    — Upgrade documentation (Word document)

NOTEBOOK EXECUTION ORDER
------------------------
1. SecureSpeak_Blackbook_WithSave.ipynb        (Master — run first, trains + saves all models)
2. SecureSpeak_Phase1_Audit.ipynb              (Dataset health check)
3. SecureSpeak_Phase2_1_PhishTank.ipynb        (Download external test data — run once)
4. SecureSpeak_NotebookA2_PhishTankFull.ipynb  (Find 194 BORDERLINE phishing URLs)
5. SecureSpeak_NotebookB_BorderlineEval.ipynb  (Real BORDERLINE evaluation — headline result)
6. SecureSpeak_Step3_SMSComparison.ipynb       (SMS vs BanglaBERT/XLM-RoBERTa/mBERT)
7. SecureSpeak_Step4_ConformalPrediction.ipynb (Conformal prediction uncertainty)
8. SecureSpeak_Step5_URLBaselines.ipynb        (URL vs URLTran comparison)

KEY RESULTS
-----------
BORDERLINE detection:  ECAFN 100% vs CATF 9.6% (real PhishTank + SCAREWARE data)
URL classifier:        97.14% vs URLTran 90.92% (external PhishTank + Tranco)
SMS classifier:        98.20% (vs BanglaBERT 99.46% — honest 1.26pp gap reported)
Conformal coverage:    92.8% empirical at 90% target (alpha=0.10)
Real-world FPR:        0.00% on 52,118 PCAPdroid Bangladesh flows

DATASETS NEEDED (on Google Drive in cse498R/Datasets/)
-------------------------------------------------------
- StealthPhisher2025.csv
- BangalaBarta bangla_spam_sms smishing.csv
- CICMalAnal2017/ (folder)
- pcapdroid_merge.csv
- PCAPdroid_zeroday_test_cleandata.csv
- PhiUSIIL_Phishing_URL_Dataset.csv (optional — used only in NotebookA which is excluded)
