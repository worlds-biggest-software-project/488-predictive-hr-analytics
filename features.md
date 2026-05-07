# Predictive HR Analytics — Feature & Functionality Survey

> Candidate #488 · Researched: 2026-05-07

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Visier People | Standalone people analytics platform | Commercial SaaS | https://www.visier.com/products/visier-people/ |
| Workday People Analytics | Embedded analytics within Workday HCM suite | Commercial SaaS (add-on) | https://www.workday.com/en-us/products/analytics-reporting/augmented-analytics.html |
| Workday Peakon Employee Voice | Continuous listening and attrition prediction | Commercial SaaS (add-on) | https://www.workday.com/en-us/products/employee-voice/overview.html |
| SAP SuccessFactors Workforce Analytics | Embedded analytics within SuccessFactors HCM suite | Commercial SaaS (add-on) | https://www.sap.com/products/hcm/workforce-analytics.html |
| Eightfold AI | AI-native talent intelligence platform | Commercial SaaS | https://eightfold.ai/ |
| Culture Amp | Employee engagement and performance platform | Commercial SaaS | https://www.cultureamp.com/ |
| Lattice | Performance management and people analytics | Commercial SaaS | https://lattice.com/platform/analytics |
| IBM Watson Talent (Kenexa) | AI-powered talent analytics suite | Commercial SaaS | https://www.ibm.com/watson/uk-en/talent/insights/ |
| OrgVue | Organisational design and workforce planning | Commercial SaaS | https://www.orgvue.com/ |
| One Model | Independent people analytics data platform | Commercial SaaS | https://www.onemodel.co/ |

---

## Feature Analysis by Solution

### Visier People

**Core features**
- Flight risk / attrition prediction with individual-level probability scores
- 50+ out-of-the-box workforce metrics (turnover, retention, diversity, productivity)
- Benchmarking against industry and peer-group data
- Succession planning and talent pipeline dashboards
- Workforce demand forecasting and scenario modelling
- Diversity, equity, and inclusion analytics
- Compensation and pay equity analysis
- Vee generative AI assistant and Vee Boards for C-suite storytelling
- Pre-built HRIS connectors (Workday, SAP, Oracle, ADP)
- Embedded NLP for natural-language querying of workforce data

**Differentiating features**
- Vee AI assistant produces narrative "insight boards" targeted at the C-suite, framing data in business-outcome language rather than HR metrics
- Scales confidently to organisations with 300,000+ employees
- Large library of pre-built benchmarks drawn from multi-tenancy data across customers

**UX patterns**
- Executive-focused visual storytelling with curated "insight boards"
- Progressive disclosure: summary cards expand to detail views
- Role-based dashboards surface role-relevant insights (CHRO vs. HRBP vs. manager)

**Integration points**
- REST API for custom data ingestion
- Pre-built connectors to 50+ HRIS, payroll, and survey platforms
- Embedded analytics available inside Workday and SAP interfaces

**Known gaps**
- High implementation cost and complexity (8–16 weeks typical)
- Requires large workforce (1,000+ employees) for individual-level predictions to be reliable
- Limited ability to customise the prediction models themselves; customers must accept Visier's model logic
- No native prescriptive action layer (recommendations stay at the data level)
- Self-service BI limited; heavily template-driven

**Licence / IP notes**
- Proprietary SaaS; no open-source components. Pricing not publicly disclosed; enterprise contracts typically $100,000–$500,000+ per year. No known patented algorithmic features, though the platform IP is wholly proprietary.

---

### Workday People Analytics

**Core features**
- Automated Storyteller Engine identifies trends and anomalies across Workday HCM data
- 40+ pre-built visualisations and 70+ HR metrics out of the box
- Illuminate AI layer for anomaly detection, predictive attrition signals, and NLP queries
- Prism Analytics for ingesting and blending external data (survey results, market compensation, financial budgets)
- Diversity, equity, and inclusion analysis including VIBE Index
- Pay equity auditing
- Skills gap analysis integrated with Workday Skills Cloud
- Manager-level dashboards surfacing team health signals

**Differentiating features**
- Automated Storyteller: scans millions of data combinations to surface the most significant workforce trends without manual querying
- VIBE Index: proprietary diversity and belonging measurement framework
- Deep integration with Workday's full HCM suite means zero data movement for Workday customers

**UX patterns**
- Narrative-first interface: insights are delivered as written stories, not raw charts
- Embedded directly within Workday HCM, reducing context-switching for HR users
- AI generates plain-language summaries for every trend identified

**Integration points**
- Native to Workday HCM — integrates automatically with Recruiting, Learning, Compensation, and Performance modules
- Prism Analytics allows REST/CSV ingestion of external data sources
- Workday-hosted data lake with governed schema management

**Known gaps**
- Only useful to organisations already on Workday HCM; creates deep vendor lock-in
- Prism Analytics for external data blending is complex to configure and requires technical expertise
- Limited ability to query ad hoc or explore data outside pre-built storylines
- Predictive models are black-box; no user-facing model explainability layer

**Licence / IP notes**
- Proprietary; bundled with or added on to Workday HCM licences. Estimated $50–$150 per employee per year for the full analytics stack.

---

### Workday Peakon Employee Voice

**Core features**
- Continuous pulse surveys (weekly, bi-weekly, or monthly cadence) with adaptive question sets
- Individual and segment-level attrition risk prediction (0–100% probability score)
- NLP-powered sentiment analysis of open-text survey responses
- Benchmarking against industry and Peakon global dataset
- 85 engagement and wellbeing metrics tracked per employee
- Manager-facing action recommendations based on team survey results
- Survey delivery via email, SMS, Slack, and Microsoft Teams
- Confidential two-way dialogue between employees and managers within the platform

**Differentiating features**
- Attrition prediction fused with engagement signal — uses survey response patterns, not just HR metadata, as the primary predictive input
- Adaptive question sets that evolve based on prior responses to avoid survey fatigue
- Anonymous dialogue channel preserves psychological safety while enabling follow-up

**UX patterns**
- Manager dashboard designed for non-HR users with action prompts ("3 people on your team are at risk — here's what to do")
- Mobile-first survey experience optimised for frontline and deskless workers
- Heat maps of engagement by team, location, and demographic segment

**Integration points**
- Native integration with Workday HCM for combining engagement signals with HR data
- Webhook and API for pushing attrition risk scores to third-party systems
- HRIS connectors to BambooHR, Greenhouse, and other non-Workday systems

**Known gaps**
- Attrition model relies heavily on survey participation; accuracy degrades with low response rates
- Predictive model is opaque — HR teams cannot inspect which features drive an individual's risk score
- Limited prescriptive depth; recommendations are generic ("have a 1:1") rather than tailored
- No compensation benchmarking integration to surface pay-driven flight risk

**Licence / IP notes**
- Proprietary SaaS; acquired by Workday in 2021. Pricing not disclosed; available as a Workday add-on.

---

### SAP SuccessFactors Workforce Analytics

**Core features**
- 2,000+ pre-built HR metrics covering headcount, attrition, diversity, performance, and compensation
- Industry benchmarking across SAP's global customer base
- Predictive attrition and succession gap modelling
- Workforce planning and scenario simulation
- People Story: auto-generated narrative summaries of key workforce trends
- Integration with SAP Analytics Cloud for advanced self-service BI
- Compliance and regulatory reporting (EEO, pay gap reporting)
- Skills ontology linked to SAP's global skills taxonomy

**Differentiating features**
- Breadth of 2,000+ metrics dwarfs competitors' out-of-the-box metric libraries
- Embedded within SuccessFactors HCM means no data integration overhead for SAP customers
- People Story feature automatically generates written narrative alongside dashboards

**UX patterns**
- Tile-based dashboard design familiar to SAP users
- Guided analytics wizards for common HR questions
- Deep drill-down capability from company-level to individual-level data

**Integration points**
- Native to SAP SuccessFactors HCM ecosystem
- SAP Analytics Cloud provides additional BI layer with custom modelling
- Partner ecosystem of implementation and data integration firms (Deloitte, Accenture, etc.)

**Known gaps**
- User experience criticised as dated and complex compared to newer-generation platforms
- Implementation cost and timeline for full suite is substantial ($100k–$2M+)
- Predictive models not easily customisable; SAP controls the model logic
- AI explanations (People Story) are often surface-level and not truly predictive
- Performance degrades for organisations with complex multi-country data structures

**Licence / IP notes**
- Proprietary; $6–$38 per user per month depending on modules. Full HCM suite enterprise contracts exceed $2M for large multinationals.

---

### Eightfold AI

**Core features**
- Deep-learning career graph built on 1.6 billion+ career profiles and 1.6 million+ skill tags
- Skills-based candidate matching (talent acquisition) and internal mobility recommendations
- Succession candidate identification and bench strength analysis
- Real-time workforce skills inventory mapped against role requirements
- AI Interviewer autonomous agent for candidate screening
- Bias reduction via demographic anonymisation during screening
- Diversity and inclusion analytics across talent acquisition and internal mobility
- Career path modelling for individual employees ("if you learn X, you could move to Y")
- Job Intelligence Engine for defining roles using market data

**Differentiating features**
- Skills graph with 1.6 billion profile data points is the largest in the market, giving more accurate skill-to-role matching than any competitor
- Autonomous AI agents (Interviewer) move beyond analytics into action
- Unified platform spanning talent acquisition, internal mobility, and succession in a single skills graph

**UX patterns**
- Employee-facing career portal for self-directed development planning
- Recruiter-facing matching interface with ranked candidate lists and explainability cards
- Manager dashboard for talent pipeline and succession bench

**Integration points**
- ATS connectors: Workday, SuccessFactors, Greenhouse, Lever, iCIMS
- HRIS connectors for employee record and performance data ingestion
- LMS connectors to recommend learning pathways
- REST API for custom integrations

**Known gaps**
- Primarily a talent acquisition and mobility tool; weaker on pure HR analytics (no cohort-level trend analysis, limited compensation analytics)
- Large-enterprise focused; SMB pricing and onboarding not viable
- Skills taxonomy maintenance requires ongoing curation by HR teams
- Model predictions are not fully explainable to individual employees

**Licence / IP notes**
- Proprietary SaaS; enterprise pricing, not disclosed publicly. Deep learning models trained on proprietary career profile dataset — potential moat but not a patent concern for open-source alternatives.

---

### Culture Amp

**Core features**
- Customisable employee engagement surveys (pulse, lifecycle, annual)
- Predictive retention analytics identifying employees at attrition risk
- Real-time driver analysis identifying top cultural and managerial drivers of engagement
- AI-powered sentiment analysis and thematic clustering of open-text responses
- Focus Agent: AI-prioritised action recommendations for managers and HR
- Performance reviews, 360 feedback, and goal tracking
- Industry benchmarking across Culture Amp's global customer database
- Heatmapping of engagement sentiment across teams, roles, and locations

**Differentiating features**
- Focus Agent: AI ranks the top 3 actions an HR team or manager should take, framing data as a prioritised to-do list rather than a dashboard to explore
- People science team embedded in product — behaviour science principles built into survey design and interpretation
- Benchmarking dataset spans 6,500+ organisations

**UX patterns**
- Survey-first UX; analytics are presented in context of survey cycle results
- Manager-facing summaries that require no data literacy to act on
- Employee-facing transparency: individuals can see aggregate results for their team

**Integration points**
- HRIS connectors: Workday, BambooHR, HiBob, Rippling, Personio
- Slack and Microsoft Teams for survey delivery and nudge notifications
- Zapier and REST API for custom integrations
- Acquired Orgnostic (2024) — integrating people analytics data mesh capabilities

**Known gaps**
- Attrition prediction relies almost entirely on survey signals; without HRIS data blending, it misses compensation and performance factors
- No native compensation benchmarking or pay equity analysis
- Limited workforce planning and headcount forecasting
- Goal-setting and OKR tooling less mature than dedicated performance management platforms

**Licence / IP notes**
- Proprietary SaaS; pricing not publicly disclosed. Typically $5–$14 per employee per month for the Engage product.

---

### Lattice

**Core features**
- Continuous performance reviews, calibration, and promotion workflows
- 1:1 meeting management and structured feedback
- Goal and OKR tracking with progress dashboards
- AI Agent: analyses data across Engagement, Performance, Grow, 1:1s, and Feedback modules to surface trends
- Succession planning and talent pipeline management
- Performance improvement plan (PIP) management
- Engagement surveys with driver analysis
- Unified analytics platform blending performance, engagement, and talent data
- Adoption Dashboard for measuring utilisation of HR programs

**Differentiating features**
- AI Agent synthesises signals across all Lattice modules (performance, engagement, goals, feedback) into a unified trend view — rare among specialist tools
- Calibration workflow with AI-generated summaries reduces time-to-consensus in promotion cycles
- HRIS module (Lattice HRIS) launched in 2023 to close the data loop

**UX patterns**
- Employee-first: strong self-service for employees to track goals, request feedback, and view their development plans
- Manager-facing dashboards for team health and performance patterns
- HR admin view for cycle management, calibration, and compliance

**Integration points**
- HRIS connectors: Workday, BambooHR, Rippling, ADP
- Slack and Microsoft Teams integration for feedback and 1:1 nudges
- Greenhouse and Lever ATS connectors
- REST API for data export

**Known gaps**
- Predictive analytics are nascent compared to dedicated platforms like Visier; trend analysis but limited forward-looking prediction
- Compensation analytics and benchmarking absent
- Workforce planning (headcount forecasting, scenario modelling) not available
- Better suited to knowledge-worker organisations; limited utility for deskless or frontline workforces

**Licence / IP notes**
- Proprietary SaaS; pricing from approximately $11–$15 per person per month depending on modules.

---

### IBM Watson Talent (Kenexa)

**Core features**
- AI-driven competency management and career development planning
- Succession planning and leadership assessment
- Workforce analytics with Watson-powered predictive insights
- Retention and compensation optimisation recommendations
- Automated HR approvals and routine data update automation
- Natural language query for workforce data exploration
- Learning recommendation engine tied to career paths
- Benefits and services Q&A chatbot

**Differentiating features**
- Deep integration with IBM's broader AI and cloud infrastructure (IBM Cloud, watsonx)
- Strong competency framework capabilities built from Kenexa's heritage as a psychometric assessment firm
- Enterprise-grade security and compliance posture suited to regulated industries

**UX patterns**
- Enterprise B2B UX; complex configuration but high customisability
- Integrated with SAP, Oracle, and Workday via IBM's middleware layer
- Primarily HR-admin facing; limited self-service for employees or managers

**Integration points**
- IBM Cloud middleware provides connectors to major HRIS platforms
- SAP SuccessFactors partner ecosystem integration
- REST APIs for enterprise data integration

**Known gaps**
- Platform has seen reduced investment relative to Visier, Workday, and Eightfold — perceived as legacy in the people analytics market
- UX is dated; significant configuration effort required before value is realised
- Predictive models less transparent than newer AI-native competitors
- Shrinking market share as customers migrate to cloud-native competitors

**Licence / IP notes**
- Proprietary SaaS/on-premise hybrid; enterprise pricing. IBM's extensive patent portfolio covers some AI methodologies but these are not specifically an obstacle for open-source development in this domain.

---

### OrgVue

**Core features**
- Organisational design modelling and visualisation
- Workforce scenario planning (M&A integration, restructuring, agile team design)
- Position and role cost modelling
- Headcount forecasting and span-of-control analysis
- Workforce transformation planning (skills needed vs. skills present)
- Integration of people data with financial and operational data
- Reporting packs for board-level organisational presentations
- Post-M&A integration workforce modelling

**Differentiating features**
- Purpose-built for organisational design decisions — fills a niche between HR analytics and management consulting tools
- Allows modelling of "what-if" organisational structures and calculating the cost/capability impact before implementing changes
- Used by management consultants (Deloitte, PwC, McKinsey) as well as HR teams

**UX patterns**
- Visualisation-heavy: org charts, network diagrams, heatmaps
- Scenario comparison view for side-by-side analysis of alternative structures
- Designed for strategic HR and transformation programme teams, not operational HR

**Integration points**
- Data import via CSV, API connectors to Workday and SAP
- Financial data ingestion from SAP FI and Oracle Finance
- Export to PowerPoint and Excel for board reporting

**Known gaps**
- Not a predictive analytics tool in the traditional sense; no flight risk or individual-level prediction
- Limited engagement and sentiment analysis capability
- Requires significant data preparation effort before modelling is possible
- High price point; primarily accessible to large enterprises and consulting firms

**Licence / IP notes**
- Proprietary SaaS; enterprise pricing not disclosed.

---

### One Model

**Core features**
- Data Mesh: automated multi-source data ingestion and normalisation from any HRIS, ATS, or survey tool
- One AI: predictive analytics layer with attrition risk, flight risk, and workforce trend forecasting
- Data Stories: visual narrative dashboards for workforce reporting
- One AI Assistant: conversational NLP interface for asking workforce questions in plain language
- Universal API Connector for integrating any data source
- Pre-built connectors to Workday, SAP SuccessFactors, Qualtrics, Greenhouse
- Scales to 7+ terabytes and 20 billion+ records per customer
- AI-powered onboarding to accelerate deployment

**Differentiating features**
- Vendor-agnostic: unlike Workday and SAP, One Model is explicitly designed to work across any HRIS stack, making it the preferred choice for multi-system organisations
- Data Mesh architecture is the most mature in the independent people analytics category
- Enterprise-grade data volume handling (7TB+, 20B+ records) outperforms most competitors

**UX patterns**
- Data-practitioner-friendly: strong tooling for HR data engineers alongside executive dashboards
- Conversational AI assistant reduces the need for technical SQL skills for HR analysts
- Modular product allows customers to start with data integration and layer analytics on top

**Integration points**
- 100+ pre-built HRIS, ATS, payroll, and survey connectors
- Universal API Connector with intelligent configuration
- REST API for custom data export

**Known gaps**
- Less known than Visier and Workday in the enterprise market; smaller benchmark dataset
- Predictive models require data maturity that many customers have not yet achieved
- AI storytelling less polished than Workday's Storyteller Engine
- Less prescriptive action layer compared to Culture Amp or Workday Peakon

**Licence / IP notes**
- Proprietary SaaS; pricing not disclosed.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Attrition/flight risk scoring at individual or cohort level
- 40–2,000+ pre-built HR metrics (headcount, turnover, diversity, compensation)
- Pre-built connectors to major HRIS platforms (Workday, SAP, BambooHR)
- Dashboard visualisations with team/department/location segmentation
- Industry benchmarking against peer organisations
- Diversity, equity, and inclusion analytics
- Role-based access control (CHRO vs. HRBP vs. manager vs. employee views)
- Export to common formats (CSV, PDF, PowerPoint) for board reporting

### Differentiating Features
- Generative AI narrative summaries (Workday Storyteller, SAP People Story, Visier Vee Boards)
- Autonomous AI agents that take action beyond surfacing data (Eightfold AI Interviewer)
- Skills graph and skills-based prediction at scale (Eightfold)
- Employee-facing transparency and self-service career portals (Eightfold, Lattice)
- Continuous listening fused with HR metadata for attrition prediction (Peakon)
- Organisational design scenario modelling (OrgVue)
- Vendor-agnostic data mesh architecture (One Model)

### Underserved Areas / Opportunities
- **Prescriptive action layer**: Most platforms surface risk scores but stop short of recommending specific, prioritised next actions with tracking of whether those actions were taken and whether outcomes improved. Culture Amp's Focus Agent is the closest but still generic.
- **Compensation benchmarking integration**: Flight risk tools rarely incorporate real-time market salary data (Radford, Mercer, Levels.fyi) as a predictive feature, yet pay-gap-from-market is a top attrition driver.
- **Model explainability for individual employees**: Platforms explain scores to HR teams but almost none provide employees with a plain-language account of what factors are being considered about them. This is an emerging legal requirement under the EU AI Act.
- **Small and mid-market access**: All leading platforms require 500–1,000+ employee datasets to produce reliable predictions. No platform has solved cohort-level or role-level prediction as a genuine product for sub-500-employee organisations.
- **Cross-domain causal inference**: Platforms correlate engagement with attrition but rarely model causal chains (e.g., manager change → engagement drop → attrition three months later). Causal AI approaches are an open research problem not yet productised.
- **Custom model configurability**: HR teams cannot inspect or adjust model features; they must accept vendor-defined prediction logic, which creates trust and regulatory concerns.
- **Action outcome tracking**: No platform systematically tracks whether HR interventions triggered by predictions (e.g., a retention conversation, a pay adjustment) actually changed the outcome and feeds that signal back into the model.

### AI-Augmentation Candidates
- **Flight risk narrative generation**: LLMs can synthesise the top 3–5 factors driving an individual's risk score into a manager-readable paragraph, replacing static dashboards with conversational summaries.
- **Survey open-text analysis**: NLP clustering and sentiment labelling of qualitative survey responses is still largely rule-based in most platforms; transformer-based models significantly outperform this.
- **Compensation gap alerting**: LLM agents can continuously monitor market salary feeds and flag individuals whose compensation has drifted below market percentile thresholds, triggering proactive retention interventions.
- **Succession reasoning**: Given a role and a bench of candidates, an LLM agent can draft a plain-language rationale for each candidate's readiness, incorporating 360 feedback, performance trajectory, and skill gap narrative.
- **Regulatory compliance drafting**: AI can auto-generate the documentation HR teams need to satisfy requests from employees under GDPR Article 22 (right to explanation of automated decisions).

---

## Legal & IP Summary

All tools surveyed are proprietary commercial SaaS products; none release source code or model weights. No patents specifically covering predictive HR analytics algorithms were identified in public patent databases, though IBM, SAP, and Workday hold broad portfolios covering AI and data analytics methods that could theoretically impinge on very specific implementations. Open-source development of a predictive HR analytics platform faces no specific IP barrier from these vendors provided it does not replicate proprietary data assets (e.g., Eightfold's 1.6 billion career profile database). The key IP risk areas are: (1) using vendor-specific terminology in product naming, (2) replicating the exact methodology of a patented IBM Watson scoring algorithm, and (3) benchmark data — industry benchmarks are derived from multi-tenancy customer data and are not reproducible without similar data scale. No copyright, trademark, or licence concerns were identified for building an open-source alternative in this space.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Multi-source HRIS data ingestion with a canonical HR data model (employee, role, event, performance)
- Attrition/flight risk model with cohort-level predictions for sub-500-employee organisations and individual-level for larger ones
- Explainability layer: top 3 contributing features per prediction in plain language
- Role-based dashboards (HR admin, manager, employee)
- Fairness audit module: demographic parity checks on all model outputs
- Pre-built connectors for at least Workday, BambooHR, and CSV import

**Should-have (v1.1)**
- Compensation benchmarking integration (Levels.fyi API or CSV ingestion of Radford/Mercer data)
- Prescriptive action recommendations with action tracking and outcome feedback loop
- Continuous pulse survey module with engagement-to-attrition signal fusion
- Workforce capacity forecasting (headcount by department, 6–18 months)
- Natural language query interface (LLM-powered "ask a question about your workforce")

**Nice-to-have (backlog)**
- Causal inference modelling for understanding which interventions change outcomes
- Employee-facing self-service view showing the factors used in their own assessment
- Succession planning bench visualisation and readiness scoring
- Organisational design scenario modelling (restructuring cost/capability modelling)
- Regulatory compliance documentation generator (GDPR Article 22 response drafts)
