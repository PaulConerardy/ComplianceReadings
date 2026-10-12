# ML/AI Detection of Trading Ahead of MNPI — Literature Summary with Links

A structured reading list for machine-learning approaches to detecting insider trading / trading ahead of price-sensitive events, covering supervised, unsupervised, graph-based, NLP, and microstructure methods.

---

## 1. Core Reviews & Regulator Baselines

### Priyadarshi & Kumar (2024) — *A comprehensive review on insider trading detection using artificial intelligence*
Systematic literature review of ML/DL for insider trading detection; covers regulatory landscape, datasets, and open challenges (data heterogeneity, cross-country regulatory variation, need for standardized datasets).
- https://link.springer.com/article/10.1007/s42001-024-00284-5
- DOI: [10.1007/s42001-024-00284-5](https://doi.org/10.1007/s42001-024-00284-5)

### Tiwari, Ramampiaro & Langseth (2021) — *Machine Learning in Financial Market Surveillance: A Survey*
IEEE Access survey of ML across market-abuse types (spoofing, insider trading, manipulation); good taxonomy of feature families and model classes.
- https://doi.org/10.1109/ACCESS.2021.3130843

### Maheshwari & Chatnani (2023) — *Applications of AI and ML-based Supervisory Technology in Financial Markets Surveillance: A Review*
SLR of 55 papers (1999–2022) on SupTech — how regulators themselves use ML.
- https://exa.ai/library/publication/v8plsjh67hw

### FCA Market Watch 79 (2024) — surveillance model failures and testing expectations
Real-world failure modes: dead news feeds, buggy alert logic, missing venue data. The practitioner baseline every model should be tested against.
- https://www.fca.org.uk/publications/newsletters/market-watch-79

### FCA Potentially Anomalous Trading Ratio (PATR) methodology
The account-level "unusual for the account / directional / profitable" logic most bank models copy.
- https://www.fca.org.uk/data/market-cleanliness-statistics-2019-20/potentially-anomalous-trading-ratio-2019-20
- MCS revision (event-study / CAR methodology): https://www.fca.org.uk/publication/research-notes/revision-market-cleanliness-statistic-methodology.pdf
- Market cleanliness statistics hub: https://www.fca.org.uk/data/market-cleanliness-statistics-2024-25

---

## 2. Supervised Classification (Enforcement-Labeled)

### Neupane & Griva (2025) — *A Random Forest Approach to Detect and Identify Unlawful Insider Trading*
110 features (ownership, governance, profitability, risk, market returns) from EDGAR Form 4 + CRSP + Compustat; ~4k labeled transactions; 96.4% accuracy. Key finding: corporate-governance and ownership features dominate pure market features.
- https://link.springer.com/article/10.1007/s10614-025-10993-2
- arXiv preprint: https://arxiv.org/html/2411.13564v1
- DOI: [10.1007/s10614-025-10993-2](https://doi.org/10.1007/s10614-025-10993-2)

### Deng, Chang & Zhao (2021) — XGBoost + NSGA-II for insider trading identification
The benchmark most later supervised work compares against (referred to as "DCZ").
- Cited via Neupane & Griva references; original: Deng et al., *An intelligent system for insider trading identification in Chinese security market*

### Li, Li, Wang & Zhang (2022) — *Identification of Insider Trading Based on Multi-task Deep Neural Network*
Multitask DNN treating each industry as a subtask (shared hidden layers + task-specific heads); 90.6% recall; SMOTE for class imbalance; multi-source features (stock performance, governance, financials, media).
- https://pmc.ncbi.nlm.nih.gov/articles/PMC9054416/
- DOI: [10.1155/2022/4874516](https://doi.org/10.1155/2022/4874516)

### Batebi (2025) — *Can Machines Better Predict Insider Trading?* (PhD dissertation, UTRGV)
LASSO / Random Forest / XGBoost with Bayesian hyperparameter tuning + SHAP; +150% R² vs OLS on magnitude; 82% accuracy / 95% recall on likelihood; Gaussian Thompson Sampling to test superior-information vs contrarian hypotheses.
- https://scholarworks.utrgv.edu/cgi/viewcontent.cgi?article=2741&context=etd

### Liu, Mai, Shan & Wu (2020) — *Predicting shareholder litigation on insider trading from financial text: An interpretable deep learning approach*
Interpretable deep learning on financial text; litigation as a labeling mechanism.
- DOI: [10.1016/j.im.2020.103387](https://doi.org/10.1016/j.im.2020.103387)

---

## 3. Unsupervised Anomaly Detection (Label-Free)

### Mazzarisi, Ravagnani, Deriu, Lillo, Medda & Russo (2024) — *A machine learning approach to support decision in insider trading detection*
Flagship Consob-collaboration paper. Two methods on full investor-resolved Italian equity data around takeover bids:
1. Dynamic k-means clustering → behavioral discontinuities before price-sensitive events (PSEs)
2. Statistically Validated Networks (SVN) → synchronized clusters = candidate insider rings
- https://link.springer.com/article/10.1140/epjds/s13688-024-00500-2
- arXiv: https://arxiv.org/abs/2212.05912
- DOI: [10.1140/epjds/s13688-024-00500-2](https://doi.org/10.1140/epjds/s13688-024-00500-2)

### Ravagnani et al. (2024) — *Dimensionality reduction techniques to support insider trading detection*
Companion to the above: PCA + autoencoders on full daily position time series; reconstruction error = anomaly score; converges with k-means + SVN results.
- https://arxiv.org/html/2403.00707v2
- arXiv: https://arxiv.org/abs/2403.00707
- Consob working paper PDF: https://www.consob.it/documents/11973/4032571/fintech12.pdf/cc892fa7-816a-0da7-d2ec-8a26e64a6499

### Poutré, Chételat & Morales (2023) — *Deep Unsupervised Anomaly Detection in High-Frequency Markets*
Transformer autoencoder on limit order book subsequences; asset-independent; no prior manipulation-pattern knowledge; state-of-the-art on synthetic quote stuffing / layering / pump-and-dump.
- https://papers.ssrn.com/sol3/papers.cfm?abstract_id=4502662

---

## 4. Graph & Network Approaches

### Baltakys, Baltakienė, Heidari, Iosifidis & Kanniainen (2023) — *Predicting the Trading Behavior of Socially Connected Investors: Graph Neural Network Approach*
GNN on investor social networks (board co-memberships + family ties); predicts an investor's trades from neighbors' behavior; high predictability = suspicious; ranks investors by suspiciousness; public anonymized dataset.
- https://papers.ssrn.com/sol3/papers.cfm?abstract_id=4163635
- DOI: [10.1016/j.eswa.2023.120285](https://doi.org/10.1016/j.eswa.2023.120285)

### *Needles in a haystack: using forensic network science to uncover insider trading* (2025)
2.9M SEC Form 4 trades (2014–2024); weighted edges by temporal trade similarity (weekly linear kernel); validated against null models; closeness/eigenvector centrality + OddBall for anomalous egonets; most anomalous egonet matched a known prosecuted insider.
- https://arxiv.org/html/2512.18918v1
- arXiv: https://arxiv.org/abs/2512.18918

### Stenfors, Guo, Li, Hewage, Mere & Chen — *Shadow trading detection: A graph-based surveillance approach* (AMGIN)
Adaptive Market Graph Intelligence Network: spatio-temporal graph fusing static corporate relationships (sector, ownership, knowledge graphs) + dynamic price co-movement; motivated by SEC v. Panuwat (2024).
- https://pure.port.ac.uk/ws/portalfiles/portal/115802685/Shadow_trading_detection.pdf

### *Can Language Models Identify Shadow Trading Targets? An NLP Evaluation of SEC Enforcement Theory* (2026)
Cautionary result: LLM semantic similarity of 10-K MD&A sections does NOT predict peer abnormal returns on M&A announcements (ρ = +0.07, p = 0.37 across 30 events), despite recovering Incyte on the Panuwat fact pattern.
- https://arxiv.org/abs/2608.01322

---

## 5. NLP / LLM Approaches

### LSEG Surveillance Guide on Amazon Bedrock (industry, AWS blog)
Production LLM pipeline: Claude classifies RNS articles on a 9-point price-sensitivity scale; optimized for 100% recall on price-sensitive; chain-of-thought justifications for audit trails.
- https://aws.amazon.com/blogs/machine-learning/how-london-stock-exchange-group-is-detecting-market-abuse-with-their-ai-powered-surveillance-guide-on-amazon-bedrock/

### Hu (Rutgers) — *A deep learning approach to the detection of illegal insider trading*
TimesNet + Time-Series Transformers forecast expected stock/options volume (tweet sentiment as covariate via pyFin-Sentiment); ANOMALOUS algorithm (normalized cross-correlation) flags volume patterns matching prosecuted cases.
- https://rucore.libraries.rutgers.edu/rutgers-lib/73544/

### *Unsupervised SEC Form 4 Footnote Change Detection Using Fine-Tuned Sentence Transformer and Multi-task LSTM Networks* (2026)
Sentence-transformer embeddings of Form 4 footnotes; multi-task LSTM predicts expected embedding; material deviation = signal; 3.3M filings 2006–2025; material changes show significantly lower CAAR.
- https://dl.acm.org/doi/10.1007/978-981-92-2888-1_12

---

## 6. Time-Series, Options & Microstructure

### Chesney, Crameri & Mancini — *Detecting Informed Trading Activities in the Options Markets*
Classical statistical method: unusual OI/volume increments + unhedged + large ex-post gains; clusters before M&A / earnings / crises; more puts than calls; OTM options in turbulent times.
- https://scispace.com/pdf/detecting-informed-trading-activities-in-the-options-markets-1meiz6f836.pdf
- DOI: [10.1016/j.jbankfin.2017.05.004](https://doi.org/10.1016/j.jbankfin.2017.05.004)

### *Velocity- and Regime-Aware Detection of Intraday Options Market Manipulation, with Explainable Attribution* (2026)
Delta-velocity + autoencoder on options; HMM regime conditioning; SHAP attribution; 10/10 regulator-identified days on BANKNIFTY.
- https://arxiv.org/html/2608.05373
- arXiv: https://arxiv.org/abs/2608.05373

### Aitken, Cumming & Zhan (2015) — *Exchange trading rules, surveillance and suspected insider trading*
Cross-market evidence: more detailed exchange rules + surveillance reduce suspected-case frequency but increase profit per case.
- https://ideas.repec.org/a/eee/corfin/v34y2015icp311-330.html

---

## 7. Suggested Reading Order

1. **FCA PATR + Market Watch 79** — practitioner baseline (links in §1)
2. **Mazzarisi et al. 2024** — best end-to-end unsupervised framing, regulator-grounded
3. **Neupane & Griva 2025** — best supervised feature-engineering study
4. **Baltakys et al. 2023** — GNN on social networks
5. **Forensic network science (arXiv 2512.18918)** — ring detection with null models
6. **Stenfors et al. (AMGIN)** — shadow trading / spatio-temporal graphs (emerging problem)
7. **Priyadarshi & Kumar 2024** — systematic review tying it together
8. **Chesney et al.** — options-based informed trading (cross-asset extension)

---

*Generated for research purposes. Some arXiv preprints may differ from final published versions; prefer publisher links where available.*
