# Predictive HR Analytics

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source platform that predicts flight risk, promotion readiness, and team conflict before they become visible -- turning lagging workforce indicators into forward-looking intelligence.

Predictive HR Analytics applies machine learning to continuous workforce signals -- engagement surveys, performance ratings, communication metadata, compensation benchmarks, and tenure data -- to surface actionable predictions for HR teams and managers. It is designed for organisations that want to move from reactive, intuition-driven people decisions to evidence-based, forward-looking workforce management without paying enterprise-tier SaaS prices or accepting vendor lock-in.

---

## Why Predictive HR Analytics?

- **Enterprise incumbents are prohibitively expensive.** Visier contracts run $100,000--$500,000+ per year. SAP SuccessFactors full-suite deployments exceed $2M for large multinationals. Workday's analytics stack costs $50--$150 per employee per year as an add-on. No credible open-source alternative exists.

- **Vendor lock-in is the norm.** Workday People Analytics only works within the Workday HCM ecosystem. SAP SuccessFactors analytics require SuccessFactors. Organisations running mixed HRIS stacks (common in mid-market and post-acquisition environments) have few independent options beyond Visier and One Model, both proprietary.

- **Prediction models are black boxes.** Workday Peakon, SAP SuccessFactors, and Visier all run opaque prediction models. HR teams cannot inspect which features drive a score, cannot adjust model logic, and cannot provide employees with plain-language explanations -- an emerging legal requirement under the EU AI Act and GDPR Article 22.

- **No platform delivers a genuine prescriptive action layer.** Most tools stop at "here is the risk score." Culture Amp's Focus Agent offers generic recommendations ("have a 1:1"), but no platform systematically tracks whether recommended interventions were taken and whether outcomes improved, feeding that signal back into the model.

- **Small and mid-market organisations are underserved.** Every leading platform requires 500--1,000+ employee datasets for reliable individual-level prediction. No tool has productised cohort-level or role-level prediction as a first-class capability for sub-500-employee organisations.

---

## Key Features

### Flight Risk and Attrition Prediction

- Individual-level probability-of-departure scores updated on a configurable cadence
- Explanatory factors surfaced per individual: compensation gap, tenure, engagement trend, manager change recency
- Cohort-level and role-level prediction mode for organisations with fewer than 500 employees
- Compensation benchmarking integration with market salary data (Levels.fyi, Radford, Mercer) to flag pay-driven flight risk

### Promotion Readiness and Succession Planning

- Multi-factor readiness model combining performance trajectory, skill coverage, cross-functional experience, and peer/manager feedback
- Competency framework comparison for target roles
- Succession bench ranking with readiness scoring for critical positions
- Impact modelling for departures in key roles

### Team Health and Workforce Forecasting

- Collaboration metadata analysis (meeting frequency, response latency, cross-team interaction volume) to identify rising conflict risk
- Workforce capacity forecasting by department, location, and skill set over 6--18 month horizons
- Skill gap analysis mapping current capabilities against future role requirements
- Upskilling and hiring priority identification with links to learning pathways

### Fairness, Explainability, and Compliance

- Demographic parity checks on all model outputs across protected characteristics
- Plain-language explanation of the top contributing features for every prediction
- Human approval gate before acting on individual-level recommendations
- Employee-facing transparency: individuals can view the factors used in their own assessment
- Regulatory compliance documentation support for GDPR Article 22 right-to-explanation requests

### Prescriptive Action Layer

- Specific, prioritised recommended actions with assigned owners and deadlines
- Action outcome tracking: monitors whether interventions were taken and whether outcomes improved
- Feedback loop into the model so prediction accuracy improves as the organisation learns what works

---

## AI-Native Advantage

Transformer-based NLP significantly outperforms the rule-based approaches most incumbents use for survey open-text analysis, enabling more accurate sentiment clustering and thematic extraction. LLMs can synthesise the top contributing factors behind a risk score into a manager-readable narrative, replacing static dashboards with conversational summaries. AI agents can continuously monitor market salary feeds and proactively flag compensation drift, and can auto-generate succession readiness rationales incorporating 360 feedback, performance trajectory, and skill gaps. Causal inference modelling -- understanding which interventions actually change outcomes rather than merely correlating signals -- is an open research problem not yet productised by any incumbent, representing a meaningful frontier for an AI-native platform.

---

## Tech Stack & Deployment

- **Data ingestion**: Canonical HR ontology with semantic normalisation layer to insulate models from HRIS schema drift (Workday regularly changes field names across releases)
- **HRIS connectors**: Pre-built integrations for Workday, SAP SuccessFactors, BambooHR, HiBob, and CSV import; REST API for custom sources
- **Model requirements**: Minimum 18--24 months of longitudinal employee data for individual-level models; cohort-level predictions available with less data
- **Deployment**: Self-hosted or cloud; all employee data remains under the organisation's control
- **Collaboration metadata**: Metadata-only analysis (who communicates with whom, not content) to stay within GDPR Article 6 and equivalent frameworks

---

## Market Context

The predictive HR analytics market is driven by growing recognition that people data rivals financial data in strategic importance. The SHRM State of AI in HR 2026 Report shows 39% of HR professionals have adopted AI in their HR functions. Incumbent pricing ranges from $5--$14 per employee per month (Culture Amp, Lattice) to $100,000--$500,000+ per year (Visier) to $2M+ for full enterprise suites (SAP SuccessFactors). Primary buyers are CHROs, VP People Analytics, and HR business partners at mid-market and enterprise organisations with 500+ employees -- though a significant underserved segment exists below that threshold.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
