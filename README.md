# 📊 Power BI Semantic Model — FMCG Sales & Operations

> A production-grade Power BI Semantic Model built on top of a dbt-modelled PostgreSQL data warehouse. Covers sales, attendance, journey compliance, payroll, indents, and outstanding — with enterprise-level Row-Level Security, incremental refresh, and Bizom-synced user access control.

---

## 🏗️ Model Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        PostgreSQL — Analytics Layer (dbt)                       │
│                                                                                 │
│   Dimension Tables                        Fact Tables                           │
│   ┌─────────────────┐                    ┌──────────────────────┐               │
│   │ dim_date        │ ─────────────────► │ fact_attendance      │               │
│   │ dim_user        │ ─────────────────► │ fact_journey_log     │               │
│   │ dim_beat        │ ─────────────────► │ fact_pjp             │               │
│   │ dim_group       │ ─────────────────► │ fact_primary_sales   │               │
│   │ dim_subgroup    │ ─────────────────► │ fact_primary_indent  │               │
│   │ dim_l2_table    │ ─────────────────► │ fact_secondary_sales │               │
│   │ dim_distributor │ ─────────────────► │ fact_sec_indent      │               │
│   │ dim_division    │ ─────────────────► │ fact_payroll         │               │
│   │ dim_seller      │ ─────────────────► │ fact_manual_claims   │               │
│   │ dim_product     │ ─────────────────► │ fact_outstanding     │               │
│   │ dim_outlet      │ ─────────────────► │ fact_bizom_pri_sales │               │
│   │ dim_zone_subzone│                    └──────────────────────┘               │
│   └─────────────────┘                                                           │
│                                                                                 │
│   Hierarchy Dimension Tables              Access Control        Measures        │
│   ┌──────────────────┐  ┌──────────────────┐  ┌────────────────┐  ┌──────────┐ │
│   │ latest_dd_table  │  │latest_l1l2_table │  │dim_access_table│  │_Measure_ │ │
│   │ distributor-div  │  │ L1-L2 user       │  │ RLS rules per  │  │ table    │ │
│   │ mapping · RLS ✓  │  │ hierarchy · RLS ✓│  │ user & level   │  │DAX logic │ │
│   └──────────────────┘  └──────────────────┘  └────────────────┘  └──────────┘ │
└─────────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                     Power BI Semantic Model (.pbism)                            │
│                                                                                 │
│   ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────────────────┐ │
│   │  🔒 RLS Rules    │  │  ⚡ Incremental  │  │  📐 DAX Measures             │ │
│   │  Bizom-synced    │  │  Refresh         │  │  90+ measures across         │ │
│   │  user access     │  │  Per fact table  │  │  8 business domains          │ │
│   └──────────────────┘  └──────────────────┘  └──────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         Power BI Dashboards                                     │
│   Sales · Attendance · Journey Compliance · Payroll · Outstanding · PJP        │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 📋 Fact Tables

| Fact Table | Description | Key Dimensions |
|---|---|---|
| `fact_attendance` | Daily attendance records per user | dim_date, dim_user, dim_beat, dim_group, dim_subgroup, dim_l2_table |
| `fact_journey_log` | Outlet visit logs, call types, time spent | dim_date, dim_user, dim_beat, dim_distributor, dim_outlet, dim_zone_subzone |
| `fact_pjp` | Planned Journey Programme — beat planning | dim_date, dim_beat, dim_group, dim_subgroup, dim_l2_table, dim_user |
| `fact_primary_sales` | Primary sales invoices | dim_date, dim_distributor, dim_division, dim_group, dim_seller, dim_product, dim_zone_subzone |
| `fact_primary_indent` | Primary order indents | dim_date, dim_distributor, dim_division, dim_group, dim_seller, dim_product, dim_zone_subzone, dim_subgroup, dim_l2_table |
| `fact_secondary_sales` | Secondary (retailer) sales | dim_date, dim_beat, dim_distributor, dim_division, dim_group, dim_seller, dim_product, dim_outlet, dim_zone_subzone, dim_subgroup |
| `fact_sec_indent` | Secondary indent orders | dim_date, dim_beat, dim_distributor, dim_division, dim_group, dim_outlet, dim_zone_subzone, dim_subgroup, dim_l2_table |
| `fact_employee_payroll` | Employee payroll & cost | dim_date, dim_user, dim_distributor, dim_division, dim_group, dim_seller, dim_zone_subzone, dim_subgroup, dim_l2_table |
| `fact_manual_claims` | Manual claim settlements | dim_date, dim_distributor, dim_division, dim_group, dim_seller, dim_zone_subzone, dim_subgroup, dim_l2_table, dim_user |
| `fact_outstanding` | Distributor outstanding amounts | dim_distributor, dim_group, dim_subgroup, dim_l2_table, dim_seller |
| `fact_bizom_primary_sales` | Bizom-sourced primary sales (sub-distributor) | dim_date, dim_distributor, dim_division, dim_group, dim_seller, dim_product, dim_zone_subzone, dim_subgroup, dim_l2_table, dim_user |

---

## 📐 DAX Measures

All measures live in a single dedicated `_Measure_table` — keeping the model clean and all business logic in one place. Measures are organised into **display folders by domain**.

### 📦 Secondary Indent (`sec_indent\`)

| Measure | Formula Logic | Folder |
|---|---|---|
| `sec_indent_invoice_amount` | SUM of order value with GST, excluding rejected, ÷ 10⁵ | `sec_indent\gst` |
| `sec_indent_net_amount` | SUM of net amount, excluding rejected, ÷ 10⁵ | `sec_indent\basic` |
| `sec_indent_uob` | DISTINCTCOUNT of outlets (non-rejected) | `sec_indent` |
| `sec_indent_pc` | DISTINCTCOUNT of order IDs (non-rejected) | `sec_indent` |
| `sec_indent_lspc` | Lines per call = rows ÷ PC | `sec_indent` |
| `sec_indent_units` | SUM of total units (non-rejected) | `sec_indent` |
| `sec_indent_thruput_with_gst` | Invoice amount ÷ UOB | `sec_indent\gst` |
| `sec_indent_thruput_without_gst` | Net amount ÷ UOB | `sec_indent\basic` |
| `sec_indent_dropsize_with_gst` | Invoice amount ÷ PC × 10⁵ | `sec_indent\gst` |
| `sec_indent_dropsize_without_gst` | Net amount ÷ PC × 10⁵ | `sec_indent\basic` |

### 🛒 Secondary Sales (`sec_sales\`)

| Measure | Formula Logic | Folder |
|---|---|---|
| `sec_sales_invoice_amount` | SUM of invoice value (Active), ÷ 10⁵ | `sec_sales\gst` |
| `sec_sales_net_amount` | SUM of value without GST (Active), ÷ 10⁵ | `sec_sales\basic` |
| `sec_sales_uob` | DISTINCTCOUNT of outlets (Active) | `sec_sales` |
| `sec_sales_units` | SUM of billed units (Active) | `sec_sales` |
| `sec_sales_thruput_with_gst` | Invoice amount ÷ UOB × 10⁵ | `sec_sales\gst` |
| `sec_sales_thruput_without_gst` | Net amount ÷ UOB × 10⁵ | `sec_sales\basic` |
| `work_sec_sales_total_cost_with_nc` | SUM of scheme claim without margin (Active) ÷ 10⁵ | `sec_sales\cost` |
| `work_sec_sales_nc_cost` | NC scheme cost net of claim_multi | `sec_sales\cost` |

### 🚶 Journey Log (`jour_log\`)

| Measure | Formula Logic | Folder |
|---|---|---|
| `jour_log_tc` | COUNTROWS of journey log | `jour_log\compliance` |
| `jour_log_pc` | COUNT of PC calls | `jour_log\compliance` |
| `jour_log_avg_time_spent_in_outlet` | Average seconds → formatted MM:SS | `jour_log` |
| `jour_log_complaint_non_telephonic %` | Compliant non-telephonic ÷ TC | `jour_log\compliance` |
| `jour_log_complaint_telephonic %` | Compliant telephonic ÷ TC | `jour_log\compliance` |
| `jour_log_non_complaint_non_telephonic %` | Non-compliant non-telephonic ÷ TC | `jour_log\compliance` |
| `jour_log_non_complaint_telephonic %` | Non-compliant telephonic ÷ TC | `jour_log\compliance` |

### 🗓️ Attendance (`attendance\`)

| Measure | Formula Logic |
|---|---|
| `atd_count` | COUNTROWS of attendance |
| `atd_status` | SELECTEDVALUE of attendance status |
| `atd_start_time` | First call start timestamp (formatted) |
| `atd_dur_btw_first_last_call` | DATEDIFF first call start → last call end, formatted HH:MM |

### 🗺️ Beat Execution (`beat_execution\`)

| Measure | Formula Logic |
|---|---|
| `total_beats` | DISTINCTCOUNT of active beats |
| `pjp_planned_beats` | Beats planned in PJP |
| `planned_beats_visited` | Planned beats with a journey log entry |
| `planned_beat_coverage_%` | Planned beats visited ÷ planned beats |
| `beats_not_planned` | Total beats − planned beats |
| `beats_not_planned_1` | Active beats in selected subzone not in PJP |
| `beats_not_planned_filter` | Row-level flag for unplanned beat filter |
| `subzone_name_for_beat` | Resolves subzone name from beat via mapping |
| `zone_name_for_beat` | Resolves zone name from beat via mapping |
| `l2_name_beat` | Resolves L2 name(s) for a beat via distributor mapping |

### 🏪 Sub/Primary Sales (`sub_pri_sales\cost`)

| Measure | Formula Logic |
|---|---|
| `work_sub_sales_total_cost_with_nc` | SUM sub_claim_without_margin (Active) ÷ 10⁵ |
| `work_sub_sales_nc_cost` | NC scheme cost net of claim_multi for primary sales |

### 📍 Mapping (`mapping\`)

| Measure | Formula Logic |
|---|---|
| `beat_total_outlet` | DISTINCTCOUNT of outlets in fact_total_outlets |

> **Total: 90+ DAX measures** across sales, indent, journey compliance, attendance, payroll, beat execution, cost, and mapping domains.

---

## 🔒 Row-Level Security (RLS)

### Design Philosophy

RLS is implemented at the **semantic model layer** using a dedicated `dim_access_table` — a centralised access control table that defines exactly what each user can see. Security is enforced on the two key hierarchy dimension tables (`latest_dd_table` and `latest_l1l2_table`), which then propagates automatically to all connected fact tables through model relationships.

Access is **multi-level** — a user can be granted visibility at the `group`, `sub_group`, or `l2` level, or full access across everything. And critically, access is **Bizom-synced** — when a user is deactivated in Bizom, they lose Power BI access automatically on the next pipeline refresh with zero manual intervention.

---

### Access Levels

| Access Type | What the user sees |
|---|---|
| `all` | Full data — no row filters applied |
| `group` | Only rows belonging to their assigned group(s) |
| `sub_group` | Only rows belonging to their assigned sub_group(s) |
| `l2` | Only rows belonging to their assigned L2 territory |

A user can have multiple rows in `dim_access_table` — e.g. access to two groups and one L2 simultaneously. The RLS logic handles this with OR conditions.

---

### How It Works

```
User logs into Power BI
        │
        ▼
USERPRINCIPALNAME() → matched against dim_access_table[mail_id]
        │
        ▼
Check access_type for that user
        │
        ├── access_type = "all"       →  Full data, no filter
        │
        ├── access_type = "group"     →  Filter to assigned group(s)
        │
        ├── access_type = "sub_group" →  Filter to assigned sub_group(s)
        │
        └── access_type = "l2"        →  Filter to assigned l2_id(s)
        │
        ▼
Filter applied on latest_dd_table and latest_l1l2_table
        │
        ▼
Propagates to ALL connected fact tables via model relationships
```

---

### RLS DAX Rules

#### Applied on `latest_dd_table`

```dax
VAR CurrentUser = USERPRINCIPALNAME()

VAR UserAccess =
    FILTER (
        dim_access_table,
        dim_access_table[mail_id] = CurrentUser
    )

VAR HasAllAccess =
    COUNTROWS (
        FILTER ( UserAccess, dim_access_table[access_type] = "all" )
    ) > 0

RETURN
HasAllAccess
    || latest_dd_table[group] IN
        SELECTCOLUMNS (
            FILTER ( UserAccess, dim_access_table[access_type] = "group" ),
            "group", dim_access_table[group]
        )
    || latest_dd_table[sub_group] IN
        SELECTCOLUMNS (
            FILTER ( UserAccess, dim_access_table[access_type] = "sub_group" ),
            "sub_group", dim_access_table[sub_group]
        )
    || latest_dd_table[l2_id] IN
        SELECTCOLUMNS (
            FILTER ( UserAccess, dim_access_table[access_type] = "l2" ),
            "l2", dim_access_table[l2]
        )
```

#### Applied on `latest_l1l2_table`

```dax
VAR CurrentUser = USERPRINCIPALNAME()

VAR UserAccess =
    FILTER (
        dim_access_table,
        dim_access_table[mail_id] = CurrentUser
    )

VAR HasAllAccess =
    COUNTROWS (
        FILTER ( UserAccess, dim_access_table[access_type] = "all" )
    ) > 0

RETURN
HasAllAccess
    || latest_l1l2_table[group] IN
        SELECTCOLUMNS (
            FILTER ( UserAccess, dim_access_table[access_type] = "group" ),
            "group", dim_access_table[group]
        )
    || latest_l1l2_table[sub_group] IN
        SELECTCOLUMNS (
            FILTER ( UserAccess, dim_access_table[access_type] = "sub_group" ),
            "sub_group", dim_access_table[sub_group]
        )
    || latest_l1l2_table[l2_id] IN
        SELECTCOLUMNS (
            FILTER ( UserAccess, dim_access_table[access_type] = "l2" ),
            "l2", dim_access_table[l2]
        )
```

---

### Why Both Tables?

`latest_dd_table` (distributor-division mapping) and `latest_l1l2_table` (L1-L2 user hierarchy) are the **two central hierarchy dimension tables** that all fact tables connect through. Applying RLS on both ensures:

- Sales data (fact_primary_sales, fact_secondary_sales, etc.) filtered via `latest_dd_table`
- User/field force data (fact_attendance, fact_pjp, fact_journey_log, etc.) filtered via `latest_l1l2_table`
- No fact table is left unprotected regardless of which dimension path it uses

---

### Bizom-Synced User Deactivation

```
User deactivated in Bizom
        │
        ▼
dim_user table refreshed on next daily pipeline run
        │
        ▼
dim_user[status] = "Inactive" for that user
        │
        ▼
dim_access_table entry for that user becomes unreachable
(or is cleaned up as part of the user sync process)
        │
        ▼
USERPRINCIPALNAME() finds no matching active row in dim_access_table
        │
        ▼
All RLS filters return FALSE → user sees zero data
        │
        ▼
Power BI access effectively revoked — zero manual intervention
```

---

### Key Design Decisions

- **Centralised access table** — `dim_access_table` is the single place to manage all access. No per-report, per-dashboard security to maintain.
- **Multi-level granularity** — same model serves national heads (all), regional managers (group/sub_group), and field executives (l2) without separate models.
- **Bizom as the authority** — user active status comes from Bizom, not from Power BI or manual lists. Deactivation is automatic.
- **Hierarchy-based propagation** — RLS on two dimension tables cascades to all 11 fact tables automatically through relationships.
- **Zero dashboard-level security** — every report built on this model inherits the same security without any extra setup.

---

## ⚡ Incremental Refresh

Implemented incremental refresh on all large fact tables to keep daily pipeline refresh times fast and efficient.

### Configuration

| Fact Table | Store rows from | Refresh rows from |
|---|---|---|
| `fact_primary_sales` | Last 12 months | Last 3 days |
| `fact_secondary_sales` | Last 12 months | Last 3 days |
| `fact_sec_indent` | Last 12 months | Last 3 days |
| `fact_primary_indent` | Last 12 months | Last 3 days |
| `fact_journey_log` | Last 6 months | Last 3 days |
| `fact_attendance` | Last 6 months | Last 3 days |
| `fact_employee_payroll` | Last 12 months | Last 1 month |
| `fact_manual_claims` | Last 12 months | Last 7 days |
| `fact_outstanding` | Last 3 months | Last 1 day |

### How Incremental Refresh Works Here

```
Daily pipeline runs (GitHub Actions)
        │
        ▼
New data lands in PostgreSQL analytics layer
        │
        ▼
Power BI REST API triggers dataset refresh
        │
        ▼
Only the last N days of each fact table are re-imported
        │
        ▼
Historical partitions remain untouched
        │
        ▼
Refresh completes in minutes instead of hours
```

### Benefits Achieved
- Reduced refresh time significantly — only recent partitions reload
- Historical data stays intact — no full reload needed daily
- API-triggered automatically post-pipeline via Python script

---

## 🗂️ Fact Table Relationship Diagrams

### fact_attendance
Connected to: `dim_date` · `dim_user` · `dim_beat` · `dim_group` · `dim_subgroup` · `dim_l2_table` · `latest_l1l2_table`

### fact_journey_log
Connected to: `dim_date` · `dim_user` · `dim_beat` · `dim_distributor` · `dim_group` · `dim_outlet` · `dim_seller` · `dim_zone_subzone` · `dim_subgroup` · `dim_l2_table` · `latest_l1l2_table`

### fact_pjp
Connected to: `dim_date` · `dim_beat` · `dim_group` · `dim_subgroup` · `dim_l2_table` · `dim_user` · `latest_l1l2_table`

### fact_primary_sales
Connected to: `dim_date` · `dim_distributor` · `dim_division` · `dim_group` · `dim_seller` · `dim_product` · `dim_zone_subzone` · `dim_subgroup` · `dim_l2_table` · `latest_dd_table`

### fact_primary_indent
Connected to: `dim_date` · `dim_distributor` · `dim_division` · `dim_group` · `dim_seller` · `dim_product` · `dim_zone_subzone` · `dim_subgroup` · `dim_l2_table` · `latest_dd_table`

### fact_secondary_sales
Connected to: `dim_date` · `dim_beat` · `dim_distributor` · `dim_division` · `dim_group` · `dim_seller` · `dim_product` · `dim_outlet` · `dim_zone_subzone` · `dim_subgroup` · `dim_l2_table` · `latest_dd_table`

### fact_sec_indent
Connected to: `dim_date` · `dim_beat` · `dim_distributor` · `dim_division` · `dim_group` · `dim_outlet` · `dim_zone_subzone` · `dim_subgroup` · `dim_l2_table` · `latest_dd_table` · `latest_l1l2_table` · `dim_user`

### fact_employee_payroll
Connected to: `dim_date` · `dim_user` · `dim_distributor` · `dim_division` · `dim_group` · `dim_seller` · `dim_zone_subzone` · `dim_subgroup` · `dim_l2_table` · `latest_dd_table`

### fact_manual_claims
Connected to: `dim_date` · `dim_distributor` · `dim_division` · `dim_group` · `dim_seller` · `dim_zone_subzone` · `dim_subgroup` · `dim_l2_table` · `dim_user` · `latest_dd_table`

### fact_outstanding
Connected to: `dim_distributor` · `dim_date` · `dim_group` · `dim_subgroup` · `dim_l2_table` · `dim_seller`

### fact_bizom_primary_sales
Connected to: `dim_date` · `dim_distributor` · `dim_division` · `dim_group` · `dim_seller` · `dim_product` · `dim_zone_subzone` · `dim_subgroup` · `dim_l2_table` · `dim_user` · `latest_dd_table` · `latest_l1l2_table`

---

## 🛠️ Tech Stack

| Component | Tool |
|---|---|
| Semantic model | Power BI Semantic Model (.pbism) |
| Data source | PostgreSQL (analytics layer via dbt) |
| Measures language | DAX |
| Security | Power BI RLS + DAX USERPRINCIPALNAME() |
| Refresh trigger | Power BI REST API (Python) |
| User sync | Bizom → PostgreSQL dim_user (daily pipeline) |
| Orchestration | GitHub Actions |

---

## 📁 Repository Structure

```
├── model/
│   ├── _Measure_table.tmdl      # All 90+ DAX measures
│   ├── dim_date.tmdl
│   ├── dim_user.tmdl            # RLS applied here
│   ├── dim_beat.tmdl
│   ├── dim_group.tmdl
│   ├── dim_subgroup.tmdl
│   ├── dim_l2_table.tmdl
│   ├── dim_distributor.tmdl
│   ├── dim_division.tmdl
│   ├── dim_seller.tmdl
│   ├── dim_product.tmdl
│   ├── dim_outlet.tmdl
│   ├── dim_zone_subzone.tmdl
│   ├── fact_attendance.tmdl
│   ├── fact_journey_log.tmdl
│   ├── fact_pjp.tmdl
│   ├── fact_primary_sales.tmdl
│   ├── fact_primary_indent.tmdl
│   ├── fact_secondary_sales.tmdl
│   ├── fact_sec_indent.tmdl
│   ├── fact_employee_payroll.tmdl
│   ├── fact_manual_claims.tmdl
│   ├── fact_outstanding.tmdl
│   └── fact_bizom_primary_sales.tmdl
├── screenshots/
│   ├── fact_attendance.png
│   ├── fact_journey_log.png
│   ├── fact_manual_claims.png
│   ├── fact_outstanding.png
│   ├── fact_payroll.png
│   ├── fact_pjp.png
│   ├── fact_primary_indent.png
│   ├── fact_primary_sales.png
│   ├── fact_sec_indent.png
│   ├── fact_sec_sales.png
│   └── fact_ss_sub_sales.png
└── README.md
```

---

## ⚡ Key Features

- **11 fact tables** covering the full FMCG sales & ops lifecycle
- **90+ DAX measures** organised into 8 business domains with display folders
- **Bizom-synced RLS** — access revoked automatically when user is deactivated in Bizom
- **Incremental refresh** — only recent partitions reload daily, keeping refresh fast
- **Single measure table** — all business logic centralised, dashboards stay clean
- **Star schema design** — optimised for Power BI query performance
- **Fully automated refresh** via Power BI REST API triggered post-pipeline

---

## 📄 License

MIT License — see [LICENSE](LICENSE) for details.
