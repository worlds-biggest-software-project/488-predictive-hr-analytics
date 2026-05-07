# Standards & API Reference

> Project: Predictive HR Analytics · Generated: 2026-05-07

## Industry Standards & Specifications

### ISO Standards

**ISO 30414:2018 (updated 2025) — Human Resource Management: Guidelines for Internal and External Human Capital Reporting**
- URL: https://www.iso.org/standard/69338.html
- The only international standard that defines specific metrics and reporting guidelines for human capital, covering workforce composition, turnover, productivity, recruitment, succession planning, and skills. Any predictive HR analytics platform producing outputs for board-level reporting should align its metric definitions to ISO 30414 to ensure comparability and credibility with investors and regulators. An updated edition was released in 2025.

**ISO/IEC 42001:2023 — Artificial Intelligence Management Systems (AIMS)**
- URL: https://www.iso.org/standard/81230.html
- Establishes requirements for an AI management system using a Plan-Do-Check-Act structure analogous to ISO 27001. Directly relevant to deploying predictive HR models: requires documented risk assessments, bias testing, human oversight mechanisms, and continuous monitoring. The EU AI Act's August 2026 enforcement deadline for high-risk systems makes ISO 42001 certification an increasingly attractive compliance pathway.

**ISO/IEC 27001:2022 — Information Security Management Systems**
- URL: https://www.iso.org/standard/27001
- Foundational information security standard applicable to any system processing sensitive employee data. Predictive HR analytics platforms handling personal data must demonstrate appropriate security controls over data storage, access, transmission, and disposal.

**ISO/IEC 27701:2019 — Privacy Information Management (Extension to ISO 27001)**
- URL: https://www.iso.org/standard/71670.html
- Extends ISO 27001 to cover privacy information management, providing requirements for PII controllers and processors. Directly relevant to HR analytics platforms acting as data processors for employee personal data under GDPR and equivalent frameworks.

**IEEE 3195.1.2 — Standard for Person Ontology**
- URL: https://standards.ieee.org/ieee/3195.1.2/11273/
- Defines a formal ontology for natural persons, providing standardised terminology and data relationships for human identity, employment, and personal attributes. Relevant to building interoperable HR data models; provides a common vocabulary that can reduce integration friction between disparate HRIS sources.

---

### W3C & IETF Standards

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- Defines the authorisation protocol used by all major HRIS APIs (Workday, SAP SuccessFactors, BambooHR, HiBob). Any predictive HR analytics platform connecting to HRIS data sources must implement OAuth 2.0 Authorization Code Grant with PKCE (RFC 7636) for secure token exchange.

**RFC 7519 — JSON Web Tokens (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- JWT is the standard bearer token format used by OAuth 2.0 implementations in HR APIs. Understanding JWT structure is required for API authentication against Workday and SAP SuccessFactors endpoints.

**RFC 7617 — The 'Basic' HTTP Authentication Scheme**
- URL: https://datatracker.ietf.org/doc/html/rfc7617
- BambooHR and several mid-market HRIS APIs use HTTP Basic Authentication with an API key as the username. Required for implementing BambooHR data connectors.

**W3C PROV-DM — The PROV Data Model**
- URL: https://www.w3.org/TR/prov-dm/
- W3C standard for representing provenance of data: who produced it, when, and how it was derived. Highly relevant to predictive HR analytics for maintaining an audit trail of how model outputs were generated — a requirement under GDPR Article 22 (right to explanation) and EU AI Act transparency obligations.

**W3C JSON-LD 1.1**
- URL: https://www.w3.org/TR/json-ld11/
- JSON-based serialisation format for Linked Data. Useful for publishing HR ontologies and workforce data schemas in a form that enables semantic interoperability between HR systems and analytics platforms.

---

### Data Model & API Specifications

**HR Open Standards (HROS) — Consortium Data Specifications**
- URL: https://www.hropenstandards.org/hr-open-standards/
- GitHub: https://github.com/HROpen
- The only independent, non-profit standards body for HR data exchange, publishing JSON Schema specifications for Assessments, Benefits, Compensation, Interviewing, Recruiting, Screening, Timecard, and Wellness. The Skills Data Workgroup is developing an API-based standard for skills data exchange, directly relevant to the skill gap analysis and workforce planning features of a predictive HR analytics platform.

**OpenAPI Specification 3.1.0 (OAS 3.1)**
- URL: https://spec.openapis.org/oas/v3.1.0.html
- The de facto standard for documenting REST APIs. A predictive HR analytics platform exposing a public API should publish an OAS 3.1 specification, which achieved full JSON Schema Draft-07 compatibility and is the version required by most enterprise API management gateways. Workday, Visier, and BambooHR all publish OAS-compatible API references.

**OData v4.0 — Open Data Protocol**
- URL: https://www.odata.org/documentation/
- SAP SuccessFactors exposes its primary Employee Central API via OData v4 (with v2 still supported for legacy integrations). Any connector to SuccessFactors must implement OData query semantics ($filter, $select, $expand, $top, $skip).

**JSON Schema Draft-07 / 2020-12**
- URL: https://json-schema.org/specification.html
- Required for validating HR data payloads ingested from diverse HRIS sources. A canonical HR data model (employee, role, event, performance record) should be specified in JSON Schema to enforce type safety and catch schema drift before it reaches prediction models.

---

### Security & Authentication Standards

**OAuth 2.0 + PKCE (RFC 7636)**
- URL: https://datatracker.ietf.org/doc/html/rfc7636
- Proof Key for Code Exchange extension to OAuth 2.0. Required for secure authorisation flows where HRIS data is accessed from a server-side analytics platform without exposing client secrets. All modern HRIS API providers support or require PKCE.

**OpenID Connect 1.0 (OIDC)**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Identity layer on top of OAuth 2.0; used for single sign-on (SSO) authentication to the analytics platform itself. Enterprise HR tools are expected to support OIDC for integration with corporate identity providers (Okta, Azure AD, Google Workspace).

**NIST AI Risk Management Framework (AI RMF 1.0) — NIST AI 100-1**
- URL: https://nvlpubs.nist.gov/nistpubs/ai/nist.ai.100-1.pdf
- Voluntary US framework for managing AI risks across four functions: Govern, Map, Measure, and Manage. Covers validity, reliability, safety, security, accountability, transparency, explainability, and fairness. An open-source HR analytics platform should document its alignment to the AI RMF's MEASURE function, particularly for bias testing and explainability requirements. The 2024 Generative AI Profile (NIST AI 600-1) extends this to LLM-powered features.

**EU AI Act — Regulation (EU) 2024/1689**
- URL: https://artificialintelligenceact.eu/
- The EU AI Act classifies AI systems used for employment decisions (hiring, promotion, performance assessment, termination) as high-risk AI systems under Annex III. From August 2, 2026, high-risk systems require: mandatory risk assessments, technical documentation, bias testing, human oversight mechanisms, transparency disclosures to affected workers, and continuous monitoring. Penalties up to EUR 15 million or 3% of global annual turnover. Any predictive HR analytics product deployed in the EU or processing data of EU employees must comply.

**GDPR Article 22 — Automated Individual Decision-Making**
- URL: https://gdpr-info.eu/art-22-gdpr/
- Individuals have the right not to be subject to decisions based solely on automated processing, including profiling, that produce significant effects. HR predictions (flight risk scores, promotion readiness ratings) that influence individual HR decisions require either explicit consent, necessity for a contract, or authorisation by Union/Member State law, plus a mandatory human review step. The system must be able to provide individuals with a meaningful explanation of the logic involved.

**OWASP API Security Top 10 (2023)**
- URL: https://owasp.org/API-Security/editions/2023/en/0x00-header/
- Defines the ten most critical API security risks. Particularly relevant for an HR analytics API handling sensitive personal data: Broken Object Level Authorisation (BOLA) is the top risk and a common vulnerability in multi-tenant HR analytics platforms where one customer could access another's workforce data.

---

### MCP Server Specifications

**Model Context Protocol (MCP) — Anthropic**
- URL: https://modelcontextprotocol.io/
- An open protocol for connecting AI language models to external data sources and tools. Relevant for building LLM-powered features within a predictive HR analytics platform — for example, an MCP server could expose structured HRIS data (employee records, performance history, engagement scores) to a Claude or GPT-based assistant that answers workforce questions in natural language. MCP enables the "natural language query" feature that platforms like Visier (Vee) and One Model (One AI Assistant) implement via proprietary means.

---

## Similar Products — Developer Documentation & APIs

### Workday — REST and SOAP APIs

- **Description:** Workday provides both SOAP-based Web Services (WWS) for legacy integrations and a modern REST API for newer functionality. The REST API is the primary interface for accessing People Analytics data, worker records, and report-as-a-service (RaaS) endpoints.
- **API Documentation:** https://community.workday.com/sites/default/files/file-hosting/restapi/
- **Web Services Directory:** https://community.workday.com/sites/default/files/file-hosting/productionapi/index.html
- **SDKs/Libraries:** No official SDK; community-maintained Python and Node.js wrappers available on GitHub
- **Developer Guide:** https://www.getknit.dev/blog/workday-api-integration-in-depth
- **Standards:** REST/JSON (modern endpoints); SOAP/XML (legacy WWS endpoints); OAuth 2.0 for authentication
- **Authentication:** OAuth 2.0 Authorization Code Grant; API clients registered in Workday tenant as Integration System Users (ISUs)
- **Notes:** Workday's REST API coverage is still incomplete — many HR data objects (org structure, compensation history) remain SOAP-only. Schema changes with each Workday release require connector maintenance.

---

### SAP SuccessFactors — OData API

- **Description:** SAP SuccessFactors exposes its Human Capital Management data via OData v2 and v4 APIs, covering Employee Central, Recruiting, Performance & Goals, and Succession modules. OData v4 is the current standard; v2 remains available for legacy entities.
- **API Documentation:** https://help.sap.com/docs/SAP_SUCCESSFACTORS_PLATFORM/d599f15995d348a1b45ba5603e2aba9b/03e1fc3791684367a6a76a614a2916de.html
- **API Explorer:** https://api.sap.com/products/SAPSuccessFactors/apis/all
- **SDKs/Libraries:** SAP Cloud SDK for Java and JavaScript; community Python wrappers
- **Developer Guide:** https://www.getknit.dev/blog/developer-guide-to-get-employee-data-from-sap-success-factors-api
- **Standards:** OData v4 (primary), OData v2 (legacy); OAuth 2.0 authentication; JSON and XML response formats
- **Authentication:** OAuth 2.0 SAML Bearer Assertion; OAuth 2.0 client credentials for machine-to-machine integration

---

### BambooHR — REST API

- **Description:** BambooHR provides a REST API covering employee records, time off, benefits, goals, training, performance reviews, and company data. Widely used by mid-market organisations (50–2,000 employees) and the most accessible HRIS API for developers.
- **API Documentation:** https://documentation.bamboohr.com/reference
- **Getting Started:** https://documentation.bamboohr.com/docs/getting-started
- **SDKs/Libraries:** https://documentation.bamboohr.com/docs/sdks (official PHP SDK; community Python and Node.js libraries)
- **Developer Guide:** https://documentation.bamboohr.com/docs/api-details
- **Standards:** REST/JSON; standard HTTP methods (GET, POST, PUT, DELETE)
- **Authentication:** HTTP Basic Authentication with API key as username; no OAuth 2.0 support

---

### HiBob (Bob) — REST API

- **Description:** HiBob's API provides programmatic access to employee records, time off, documents, goals, workforce planning, and custom fields. HiBob is popular in tech-scale-up organisations (200–5,000 employees) in Europe and Asia-Pacific.
- **API Documentation:** https://apidocs.hibob.com/
- **Employee Data Guide:** https://apidocs.hibob.com/docs/explore-employee-data
- **SDKs/Libraries:** No official SDK; REST/JSON client in any HTTP library
- **Developer Guide:** https://apidocs.hibob.com/docs/getting-started
- **Standards:** REST/JSON; supports both search-based and direct employee ID endpoints
- **Authentication:** Service user credentials (API token) via HTTP Basic Authentication

---

### Visier — Data Intake and Query APIs

- **Description:** Visier provides a suite of developer APIs for ingesting HR data into the Visier platform and querying analytics results programmatically. Relevant for building an alternative platform with compatible data ingestion patterns.
- **API Documentation:** https://docs.visier.com/developer/apis/apis.htm
- **Data Intake API:** https://docs.visier.com/developer/apis/data-in/data-intake/data-intake-api.htm
- **Data Query API:** https://docs.visier.com/developer/apis/data-out/data-query/data-query-api.htm
- **SDKs/Libraries:** Postman collection available; REST/JSON client in any HTTP library
- **Developer Guide:** https://docs.visier.com/developer/Platform/overview.htm
- **Standards:** REST/JSON; OpenAPI-compatible
- **Authentication:** OAuth 2.0

---

### Merge.dev — Unified HRIS API

- **Description:** Merge provides a single unified API that normalises data from 200+ HRIS, ATS, payroll, and accounting platforms (including Workday, SAP SuccessFactors, BambooHR, HiBob, Gusto, ADP) into a single data model. Eliminates the need to build and maintain individual HRIS connectors.
- **API Documentation:** https://docs.merge.dev/hris/
- **Unified API Overview:** https://docs.merge.dev/merge-unified/hris/overview
- **SDKs/Libraries:** JavaScript, Python, Ruby, Java, Go SDKs available
- **Standards:** REST/JSON; OpenAPI 3.0 spec published
- **Authentication:** API key + account token per integration; OAuth 2.0 underlying per integration
- **Notes:** Merge is the fastest path to multi-HRIS support; commercial pricing is per integration per month. Worth evaluating as a dependency for MVP to avoid building 10+ separate HRIS connectors.

---

### Finch — Unified Employment API

- **Description:** Finch provides a unified API specifically for HRIS and payroll data, covering organisation, directory, employment, individual, pay statement, and benefits endpoints. Focused on employment data rather than the full HRIS feature set.
- **API Documentation:** https://www.tryfinch.com/finch-api
- **SDKs/Libraries:** JavaScript, Python, Java, Kotlin, Go SDKs
- **Standards:** REST/JSON; OpenAPI spec published
- **Authentication:** OAuth 2.0 with employer authorisation flow
- **Notes:** Finch's data model is more opinionated than Merge's, which makes it easier to work with but limits access to HR-specific custom fields.

---

### Culture Amp — API (Limited)

- **Description:** Culture Amp provides a limited API for accessing survey results, employee engagement data, and performance review outcomes. Primarily used by system integrators connecting Culture Amp engagement scores to HRIS and analytics platforms.
- **API Documentation:** https://developers.cultureamp.com/
- **Standards:** REST/JSON; GraphQL endpoint available for some data objects
- **Authentication:** OAuth 2.0
- **Notes:** Culture Amp's API is not fully public; access requires approval. GraphQL endpoint is more powerful than the REST API for querying engagement data flexibly.

---

## Notes

**Emerging standards to monitor:**

- **EU AI Act implementing acts (2026)**: The European Commission is developing technical standards for high-risk AI system documentation and testing, delegated to CEN-CENELEC and ETSI. These will become the operative compliance checklist for HR AI systems deployed in the EU; first implementing acts expected H2 2026.
- **ISO/IEC CD 5338 — AI Lifecycle Processes**: Draft standard for AI system lifecycle processes analogous to ISO/IEC 12207 for software. Once published, will provide a process framework for developing and maintaining predictive HR models.
- **IEEE P7003 — Algorithmic Bias Considerations**: Working group developing a standard for identifying and mitigating algorithmic bias. Directly relevant to fairness audit requirements for HR prediction models.
- **HR Open Standards Skills API**: The skills data exchange standard under development by the HR Open Standards Skills Data Workgroup will become important for the skill gap analysis and workforce planning features once ratified.

**Gap: no universal HR analytics data interchange standard exists.** Despite HR Open Standards' work, there is no widely adopted open standard for exchanging analytics results (as opposed to raw HR records) between platforms. Every vendor uses a proprietary data model for predictions, scores, and insights. This represents both an integration challenge and an opportunity for an open-source platform to establish a de facto standard.
