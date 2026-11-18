# Canada Infrastructure Bank (CIB) — Dynamics 365 & Power Platform CRM Portfolio

An end-to-end **Microsoft Dynamics 365 / Power Platform CRM** and **Executive BI** solution modeled on the investment lifecycle, data governance, and portfolio reporting workflows of the **Canada Infrastructure Bank (CIB)**.

Built to demonstrate core **CRM Business Analyst** competencies across **Dataverse Data Modeling**, **Model-Driven Apps**, **Business Process Flows (BPF)**, **Power Automate Integration & Error Logging**, **Role-Based Access Control (RBAC)**, **Application Lifecycle Management (ALM)**, **Power BI Executive Reporting**, and **UAT Quality Assurance**.

---

## 1. Executive Summary & Business Problem

Public-private infrastructure lenders manage multi-billion-dollar deal pipelines spanning clean energy, transit, broadband, trade, and Indigenous infrastructure. Relying on disconnected spreadsheets or public CMS exports creates operational bottlenecks:
* **Fragmented Deal Tracking:** No standardized stage-gate governance from initial mandate screening through Financial Close and construction monitoring.
* **Integration Blind Spots:** External portfolio accounting systems (such as **Charles River Private Markets [CRPM]**) require structured synchronization with audit trails when commitment amounts or delivery statuses change.
* **Access Governance Risks:** Analysts require write access to active deals, whereas executive reviewers and auditors require strict read-only visibility without risk of altering financial figures or audit logs.

### Solution Highlights
* **119 Real-World CIB Commitments Modeled ($20,059.5M total capital)** ingested via ETL cleansing into a normalized relational Dataverse schema.
* **5-Stage Business Process Flow** enforcing stage-gate discipline across the investment lifecycle.
* **Simulated CRPM Integration & Audit Logging** in Power Automate with automated exception handling (`Success` vs. `Failed` sync runs) and Financial Close alerting.
* **Least-Privilege Security Model** separating operational analysts from read-only reviewers, packaged in both **Unmanaged** and **Managed** ALM solution artifacts (`v1.0.0.0`).
* **Interactive Power BI Executive Report** tracking sector allocation, Indigenous infrastructure share (**15.9% / $3,185.0M across 39 projects**), delivery status, and deal size tiers.

---

## 2. System Architecture & Entity-Relationship Diagram (ERD)


### End-to-End Solution Architecture

```mermaid
flowchart LR
    A[Public CIB Dataset / CSV] -->|Power Query / Python ETL| B[(Dataverse Tables\ncib_ Publisher)]
    C[Simulated CRPM Payload\nManual / API Trigger] -->|Power Automate Flow #1| B
    B -->|Audit Trail Write| D[(cib_crpmsynclog\nSync Audit Table)]
    B -->|Stage Change Trigger| E[Power Automate Flow #2\nFinancial Close Alert]
    B -->|Model-Driven App + BPF| F[CIB Investment Tracker\nRBAC Secured UI]
    B -->|Star Schema / DAX| G[Power BI Report\nExecutive Analytics]
```

### Dataverse Relational Data Model (ERD)

```mermaid
erDiagram
    PROPONENT_PARTNER ||--o{ INFRASTRUCTURE_PROJECT : "leads (1:N)"
    INFRASTRUCTURE_PROJECT ||--o{ CRPM_SYNC_LOG : "audited by (1:N)"

    PROPONENT_PARTNER {
        guid cib_proponentpartnerid PK
        string cib_organizationname "Primary Column"
    }

    INFRASTRUCTURE_PROJECT {
        guid cib_infrastructureprojectid PK
        string cib_projectname "Primary Column"
        string cib_cibprojectid "Unique External ID"
        choice cib_prioritysector "Clean Energy, Housing, Trade, Digital"
        boolean cib_indigenousinfrastructure "Yes / No"
        choice cib_region "Western, Central, Atlantic, Northern, Canada-wide"
        choice cib_primaryprovinceterritory "Province / Territory"
        string cib_additionalprovinces "Multi-jurisdiction text"
        string cib_locationdetail "Municipality / Site"
        choice cib_participationmilestone "Commitment, Financial Close, Complete"
        choice cib_deliverystatus "Under Development, Under Construction, Complete"
        string cib_commitmenttextraw "Original scraped string"
        decimal cib_cibcommitmentm "Commitment ($M)"
        url cib_projecturl "Public project profile"
        lookup cib_leadpartnerid FK "Lookup to Proponent Partner"
        formula cib_dealsizetier "Power Fx: Mega / Large / Mid-Market"
    }

    CRPM_SYNC_LOG {
        guid cib_crpmsynclogid PK
        string cib_syncrunname "Primary Column"
        datetime cib_synctimestamp "Execution UTC timestamp"
        choice cib_syncstatus "Success, Partial Warning, Failed"
        integer cib_recordsprocessed "Count of updated records"
        string cib_errordetails "Exception payload / diagnostics"
        lookup cib_relatedprojectid FK "Lookup to Infrastructure Project"
    }
```

---

## 3. Functional Specification & ETL Data Mapping

### A. Source-to-Target ETL Mapping (`convertcsv.csv` → Dataverse)
Raw web/CMS exports contain nested array headers (`sector/0/title`), unparsed currency strings (`"$1.83 Billion \n(Under development)"`), and double-encoded UTF-8 characters (`RÃ©seau`). Prior to Dataverse ingestion and BI modeling, the dataset was normalized as follows:

| Raw Source Field | Target Dataverse Column (`cib_`) | Data Type | Transformation & Business Logic |
| :--- | :--- | :--- | :--- |
| `id` | `cib_cibprojectid` | Single Line of Text | Cast to text identifier; serves as external reconciliation key for CRPM syncs. |
| `title` | `cib_projectname` | Single Line of Text | Reversed double-encoded UTF-8 mojibake (`cp1252` → `utf-8`) to restore French and Indigenous orthography. |
| `partner` | `cib_leadpartnerid` | Lookup (`Proponent Partner`) | Extracted distinct proponent organizations into parent master table (`cib_partners_clean.csv`). |
| `sector/0/title` | `cib_prioritysector` | Choice | Mapped to 4 core CIB investment pillars (*Clean Energy*, *Enabling Housing Supply*, *Trade and Transportation*, *Digital Infrastructure and AI*). |
| `sector/1/title` | `cib_indigenousinfrastructure` | Two Options (Yes/No) | Derived boolean flag (`Yes` when secondary sector = *Indigenous Infrastructure*, else `No`). |
| `locationCategories/0/title` | `cib_region` | Choice | Mapped to *Western*, *Central*, *Atlantic*, *Northern*, or *Canada-wide*. |
| `provinces/0/title` | `cib_primaryprovinceterritory` | Choice | Primary Canadian jurisdiction; multi-province spillovers (`provinces/1..3`) concatenated into `cib_additionalprovinces`. |
| `participationCategories/0/title` | `cib_participationmilestone` | Choice | Primary deal milestone (*Commitment*, *Financial Close*, *Complete*). |
| `participationCategories/1/title` | `cib_deliverystatus` | Choice | Physical execution phase (*Under Development*, *Under Construction*, *Complete*). |
| `commitment` | `cib_cibcommitmentm` | Decimal Number | Parsed text units (`Billion` × 1000, `million` × 1) into standardized millions (`$M`); raw string preserved in `cib_commitmenttextraw`. |
| *Calculated (Power Fx)* | `cib_dealsizetier` | Formula Column | `If(Commitment >= 500, "Mega ($500M+)", If(Commitment >= 100, "Large ($100M-$499M)", "Mid-Market (<$100M)"))` |

### B. Business Process Flow: `CIB Investment Lifecycle`
Enforces standardized stage gates on the `Infrastructure Project` main form:
1. **Stage 1: Intake & Mandate Filter** — Verify `Priority Sector`, `Region`, `Primary Province`, and `Indigenous Infrastructure` eligibility.
2. **Stage 2: Appraisal & Advisory** — Validate `Lead Partner` due diligence and preliminary `Location Detail`.
3. **Stage 3: Investment Structuring** — Finalize `CIB Commitment ($M)` and verify automated `Deal Size Tier` classification.
4. **Stage 4: Final Negotiation & Closing** — Advance `Participation Milestone` to *Financial Close* (triggers automated executive notification).
5. **Stage 5: Funding, Monitoring & Exit** — Track ongoing `Delivery Status` (*Under Construction* → *Complete*) and CRPM sync health.

### C. Power Automate Integration & Event Workflows
1. **Flow #1 — `Simulate CRPM Project Sync` (Instant Cloud Flow):**
   * **Trigger:** Manual inputs (`CRPM Project ID`, `Updated Commitment ($M)`, `Updated Delivery Status`).
   * **Logic:** Queries Dataverse `Infrastructure Projects` where `cib_cibprojectid` equals the input ID.
   * **Branching & Error Handling:** If a matching project is found, updates the commitment and delivery status and writes a `Success` audit row to `CRPM Sync Log` linked to the project. If zero records match (e.g., invalid external ID), catches the exception and writes a `Failed` audit row with diagnostic details (`"No matching CIB Project ID found in Dataverse"`).
2. **Flow #2 — `Alert on Financial Close` (Automated Dataverse Trigger):**
   * **Trigger:** Row added or modified on `Infrastructure Project` filtered to attribute `cib_participationmilestone` where value equals `Financial Close`.
   * **Action:** Dispatches an executive milestone summary alert containing Project Name, Lead Partner, Sector, and CIB Commitment ($M).

---

## 4. Security Governance (RBAC) & ALM Strategy

### Role-Based Access Control (Least Privilege)
Two custom security roles were built inside the solution and bound to the **CIB Investment Tracker** Model-Driven App:

| Table / Asset | CIB Investment Analyst (Operational) | CIB Read-Only Reviewer (Audit / Exec) | Governance Rationale |
| :--- | :--- | :--- | :--- |
| **Proponent Partner** | Create/Write/Append (BU), Read/Append To (Org), **Delete: None** | Read (Org), All Others: **None** | Prevents accidental hard-deletion of master partner records; enables Lookup binding (`Append To`). |
| **Infrastructure Project** | Create/Write/Append/Append To (BU), Read (Org), **Delete: None** | Read (Org), All Others: **None** | Enables national pipeline visibility while enforcing record deactivation over deletion. |
| **CRPM Sync Log** | **Read (Org), Create/Write/Delete: None** | **Read (Org), Create/Write/Delete: None** | Integration logs are system-written by Power Automate; analysts can troubleshoot failures without altering audit history. |
| **BPF & Model-Driven App** | Create/Read/Write BPF (Org), Read App (Org) | Read BPF (Org), Read App (Org) | Analysts can advance deal stages; reviewers can inspect current stage-gate progress read-only. |

### Application Lifecycle Management (ALM)
All 9 custom components (3 Tables, 1 Model-Driven App, 1 Business Process Flow, 2 Cloud Flows, 2 Security Roles) are encapsulated under the `CIB Investment Tracker` solution (`v1.0.0.0`, Publisher prefix `cib_`):
* `solutions/CIBInvestmentTracker_1_0_0_0_unmanaged.zip` — Source-control artifact for Development environments.
* `solutions/CIBInvestmentTracker_1_0_0_0_managed.zip` — Locked release artifact for UAT/Production deployment.

---

## 5. Executive BI & Analytics (Power BI)

The Power BI report (`powerbi/CIB_Investment_Portfolio_Report.pbix`) models the cleaned CIB portfolio schema with a dedicated `_Measures` DAX table (`Total CIB Commitment ($M)`, `Total Projects`, `Average Deal Size ($M)`, `Indigenous Commitment ($M)`, and `Indigenous Portfolio Share %`).

> **Architectural Note on Data Source Portability:** To prevent trial-tenant expiration from breaking external evaluation, the `.pbix` file embeds the schema-identical `cib_projects_clean.csv` dataset with Power Query M transformations (UTF-8 mojibake repair, conditional `Deal Size Tier` logic, and numeric `Deal Size Sort` ordering). In production, the source step swaps cleanly to the native Dataverse connector.

![Power BI Executive Dashboard](screenshots/04_powerbi_executive_dashboard.png)

---

## 6. User Acceptance Testing (UAT) Test Plan

| Test ID | Scenario / Feature | Preconditions | Test Steps | Expected Result | Status |
| :--- | :--- | :--- | :--- | :--- | :---: |
| **UAT-01** | **Power Fx Deal Size Tier Calculation** | User logged into Model-Driven App with Analyst role. | 1. Open *Infrastructure Projects* → **+ New**.<br>2. Enter Project Name `"Test Solar Grid"` and `CIB Commitment ($M)` = `550`.<br>3. Save record. | `Deal Size Tier` automatically evaluates to **`Mega ($500M+)`**. Updating commitment to `250` recalculates to **`Large ($100M–$499M)`**. | Pass |
| **UAT-02** | **Relational Lookup Binding (`1:N`)** | At least one `Proponent Partner` record exists. | 1. Open an `Infrastructure Project` record.<br>2. Select a `Lead Partner` via the Lookup control and save.<br>3. Open that `Proponent Partner` record. | Project saves without privilege errors and appears in the partner's related projects subgrid. | Pass |
| **UAT-03** | **BPF Stage-Gate Progression** | Project record is in Stage 1 (*Intake & Mandate Filter*). | 1. Click Stage 1 in the *CIB Investment Lifecycle* BPF bar.<br>2. Populate required sector/region fields and click **Next Stage**. | BPF advances to Stage 2 (*Appraisal & Advisory*) and persists active stage on refresh. | Pass |
| **UAT-04** | **CRPM Sync Flow — Happy Path** | Project with `CIB Project ID` = `496` exists in Dataverse. | 1. Run Flow #1 (*Simulate CRPM Project Sync*) with ID `496`, Commitment `480`, Status `Under Construction`. | Project `496` updates to `$480M` and a new `CRPM Sync Log` record is created with `Sync Status = Success` and `Records Processed = 1`. | Pass |
| **UAT-05** | **CRPM Sync Flow — Exception Handling** | No project exists with `CIB Project ID` = `999999`. | 1. Run Flow #1 with ID `999999`, Commitment `100`, Status `Under Development`. | Flow completes gracefully without crashing; writes a `CRPM Sync Log` entry with `Sync Status = Failed` and populated `Error Details`. | Pass |
| **UAT-06** | **Automated Financial Close Alert** | Project `Participation Milestone` is currently `Commitment`. | 1. Open project record, change `Participation Milestone` to `Financial Close`, and save. | Flow #2 triggers within 60 seconds and dispatches the Financial Close summary notification. | Pass |
| **UAT-07** | **RBAC Least-Privilege Enforcement** | Security roles configured and bound to App. | 1. Inspect `CIB Investment Analyst` vs. `CIB Read-Only Reviewer` privilege matrices. | Analyst has BU Create/Write and `Delete = None`; Reviewer has `Read = Organization` and `Create/Write/Delete = None`; neither can edit `CRPM Sync Log`. | Pass |
| **UAT-08** | **ALM Solution Packaging** | All 9 components added to `CIB Investment Tracker`. | 1. Run **Publish all customizations**.<br>2. Export Unmanaged and Managed `.zip` files at `v1.0.0.0`. | Both `.zip` archives export without missing-dependency warnings and contain all tables, app, BPF, flows, and roles. | Pass |

---

## 7. Application & Governance Screenshots

| Solution Components (ALM) | Model-Driven App & BPF |
| :---: | :---: |
| ![Solution Components](screenshots/03_solution_components_alm.png) | ![Model-Driven App](screenshots/06_bpf_project_form.png) |

| Security Role: CIB Investment Analyst | Security Role: CIB Read-Only Reviewer |
| :---: | :---: |
| ![Analyst Role](screenshots/01_security_role_analyst.png) | ![Reviewer Role](screenshots/02_security_role_reviewer.png) |

| Power Automate: Simulated CRPM Sync & Audit Logging | Active Projects View |
| :---: | :---: |
| ![CRPM Sync Flow](screenshots/07_power_automate_crpm_flow.png) | ![Active Projects View](screenshots/05_model_driven_app_view.png) |