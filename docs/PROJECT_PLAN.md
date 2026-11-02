# Indigenous SCADA/EMS Suite: Detailed Project Plan

Status: draft for team review (v0.2, 2026-11-02, schema findings added)
Team: 3 developers, C++ as the main language, power-systems domain lead in the team.
Reference: EOI "Development of Indigenous SCADA/EMS Software Applications for Load Despatch Centres in India" (36 months, 3 milestones).

---

## 1. How to read this plan

The work is split into **phases**. Each phase has **work packages (WP)**, and each WP has
**tasks** small enough for one developer in 1 to 5 days. Every task ends with something
that can be checked: a document, a table, a test passing, or a demo.

Developer roles (rename to real names):

| Role | Owns | Also helps with |
|---|---|---|
| **Dev-M (Model)** | CIM database, PDS (model editor/importer), validation, topology | EMS network model build |
| **Dev-R (Real-time)** | RTDB, FEP/protocols, SCADA alarms/control, Historian, ICCP | HA/redundancy, security services |
| **Dev-A (Applications)** | Power flow, SE, CA, OPF and other EMS apps, test cases vs PSS/E | HMI displays for EMS |

The domain lead (power systems) reviews every schema change and every EMS result.

---

## 2. Big-picture architecture (one paragraph per layer)

1. **CIM Model Database (source of truth).** PostgreSQL. Holds the static network model as
   CIM classes (IEC 61970-301), plus SCADA point definitions (Measurement classes) and India
   extensions. Versioned, with audit trail. Every other app loads from here; nothing else
   owns model data.
2. **Model Build / PDS.** Imports existing DB and CIM/CGMES XML (IEC 61970-552), validates,
   runs topology and a power flow check, then "publishes" a model version to real-time.
3. **Real-Time Database (RTDB).** In-memory C++ store (shared memory), built from a published
   model version. Holds current values, quality flags and timestamps for up to 20 lakh
   analog + 20 lakh status points.
4. **Data Acquisition.** FEP (IEC 60870-5-104 first, then -101, later IEC 61850/PMU C37.118)
   and ICCP/TASE.2 (IEC 60870-6). They write to the RTDB.
5. **SCADA services.** Limit checking, alarm/event processing, calculations, control
   (select-before-operate), tagging.
6. **Historian.** Stores RTDB changes (report-by-exception + deadband + compression), trends
   and reports. See sizing note in section 8.
7. **EMS engine.** Topology processor, Power Flow, State Estimator, Contingency Analysis,
   Security Enhancement, OPF, Short Circuit, TCM, AGC, Load Forecasting, Outage Scheduler,
   DSA, DTS. Reads the network model + RTDB snapshot.
8. **HMI / Web.** Browser-based operator displays (single-line diagrams from IEC 61970-453),
   alarm list, trends. Web replica for external stakeholders sits in a separate (IT) zone.
9. **Platform services.** Message bus, HA (main/standby, main/backup control centre),
   security (RBAC, MFA, audit, IEC 62351 for comms), logging, configuration.

```
            +-----------------------+       publish model version
 PDS/Model  |  CIM Model DB (PG)    |------------------------------+
 Editor --->|  + versions + audit   |                              v
            +-----------------------+                    +------------------+
                                                         |  RTDB (in-memory)|
  RTU/SAS --IEC104--> FEP ----------------------------->|  points, values  |
  Other CC --ICCP---> ICCP server --------------------->|  quality, time   |
                                                         +------------------+
                                                          |   |    |     |
                                    SCADA alarms/control <-+   |    |     +-> HMI / Web
                                    Historian <----------------+    |
                                    EMS apps (SE, CA, OPF ...) <----+
```

---

## 3. Technology stack (proposal: each item is a decision to accept or change)

| Area | Proposal | Why |
|---|---|---|
| Language | **C++20** for core and apps; **Python** for tools/tests; **TypeScript** for web HMI | Performance where it matters; fast tooling elsewhere |
| Build | CMake + vcpkg, GCC 12+/Clang 16+ on Linux, MSVC for dev on Windows | Standard, works on Make-in-India hardware |
| OS (target) | Linux (RHEL-compatible or Ubuntu LTS) | Vendor neutral, hardened |
| Model DB | **PostgreSQL 16** | Open source, mature, good for relational CIM schema |
| Historian storage | PostgreSQL partitions for the first version, then evaluate a column store | Start simple, measure, then scale |
| Sparse maths | **Eigen** + **SuiteSparse KLU** | Fast sparse LU for PF/SE on 10,000+ buses |
| CIM XML parsing | pugixml or libxml2 | Fast RDF/XML reading for 61970-552 |
| Message bus | ZeroMQ (or NNG) inside the control centre | Simple, fast, no broker to run |
| Web/API server | Drogon or Boost.Beast (REST + WebSocket) | C++ native |
| Tests | GoogleTest (C++), pytest (Python), IEEE test cases | |
| Python bindings | pybind11 | Lets the team test C++ code from Python quickly |

Licensing note: protocol stacks (IEC 104, ICCP) and security code should be written
in-house. The EOI requires security-critical components to be indigenous, and many
open-source protocol libraries are GPL.

C++ ramp-up for the team (first 2 weeks, part time): "A Tour of C++" (Stroustrup),
the C++ Core Guidelines, and building one small tool end to end (the CSV/DB exporter in WP1.2).

---

## 4. Repository layout

```
scadaproject2026/
  docs/                    plans, architecture, decision records (docs/adr/NNN-*.md)
  schema/
    cim/                   CIM-compliant DDL (this phase)
    migrations/            numbered SQL migrations
    mapping/               old-DB -> CIM mapping sheets
  cmake/  third_party/
  src/
    common/                logging, config, time, ids (mRID/UUID)
    model/                 CIM classes in C++, DB access, CIM XML import/export
    topology/              topology processor
    rtdb/                  real-time database
    fep/  iccp/            protocols
    scada/                 alarms, events, control, calculations
    historian/
    ems/                   pf/ se/ ca/ opf/ sc/ tcm/ agc/ lf/ os/ dsa/ dts/
    hmi-server/
  web/                     browser HMI
  tests/                   unit/, integration/, cases/ (IEEE 14/118/300, synthetic only)
  tools/                   python scripts (converters, validators)
```

Rule: **no real Indian grid data in Git.** Real model and SCADA data stay on the team's
machines; the repo holds only IEEE/synthetic cases.

---

## 5. Phases and work packages

Durations assume 3 developers. Months are from project start.

### Phase 0: Foundation (weeks 1 to 3)

| # | Task | Owner | Done when |
|---|---|---|---|
| 0.1 | Create repo layout above, CMake skeleton, `hello` library + test | Dev-R | `cmake --build` and `ctest` pass |
| 0.2 | CI on GitHub Actions: build, tests, clang-format, clang-tidy | Dev-R | PRs show green/red checks |
| 0.3 | Coding standard doc (C++ Core Guidelines subset, naming, error handling) | Dev-A | docs/CODING_STANDARD.md merged |
| 0.4 | Write ADR-001 tech stack, ADR-002 "CIM DB is the single source of truth" | Domain lead | ADRs merged |
| 0.5 | Install PostgreSQL locally on all dev machines; shared dev DB on a team server | Dev-M | Everyone can connect |
| 0.6 | Collect references: current SCADA/EMS schema, CIM standards, IEEE test cases | Domain lead | Files in agreed place (not Git for real data) |

### Phase 1: CIM-compliant Model Database (weeks 3 to 14), EOI Milestone-1 core

This is the phase you start coding from.

**WP1.1 Analyse the current database (weeks 3 to 4)**

| # | Task | Owner | Done when |
|---|---|---|---|
| 1.1.1 | List every table in the current SCADA DB and EMS DB with row counts and purpose | Dev-M | `schema/mapping/current_inventory.xlsx` |
| 1.1.2 | For each table, list columns, keys, units, and which app uses it | Dev-M + Dev-A | Inventory complete |
| 1.1.3 | Mark each column: **keep (maps to CIM)**, **keep as extension**, **drop (derived/obsolete)** | Domain lead | Every column has a decision |
| 1.1.4 | Note the "best features" to keep: naming convention, point numbering, hierarchy, alarm limits, etc. | Domain lead | Short list in mapping doc |

**WP1.2 Define the project CIM profile (weeks 4 to 6)**

| # | Task | Owner | Done when |
|---|---|---|---|
| 1.2.1 | Pick the CIM version (recommend CIM16/CGMES 2.4.15 for exchange compatibility, with CIM17/CGMES 3.0 tracked) | Domain lead | ADR-003 |
| 1.2.2 | Select classes needed from IEC 61970-301 (see starter list in section 6) | Dev-M | Class list agreed |
| 1.2.3 | Map each "keep" column to a CIM class/attribute | Dev-M | `schema/mapping/old_to_cim.csv` |
| 1.2.4 | Define India extensions (namespace `in:`) for fields CIM lacks | Domain lead | Extension list with reasons |
| 1.2.5 | Define naming convention and mRID rules (UUID v4, stable across versions) | Dev-M | Section in schema doc |

**WP1.3 Write the schema (DDL) (weeks 6 to 9)**

| # | Task | Owner | Done when |
|---|---|---|---|
| 1.3.1 | Core tables: IdentifiedObject base columns, BaseVoltage, region/substation/voltage level/bay containment | Dev-M | Migration 001 applies cleanly |
| 1.3.2 | Connectivity: Terminal, ConnectivityNode, TopologicalNode | Dev-M | Migration 002 |
| 1.3.3 | Equipment: lines, transformers (+ ends, tap changers), generators, loads, shunts, switches | Dev-M + Dev-A | Migration 003 |
| 1.3.4 | Renewables, BESS, HVDC, series compensation, SVC/STATCOM | Dev-A | Migration 004 |
| 1.3.5 | Measurement model: Measurement, Analog, Discrete, Accumulator, limits, MeasurementValueSource; link to RTU/protocol addresses | Dev-R | Migration 005 |
| 1.3.6 | Model versioning, change sets, audit trail, jurisdiction (NLDC/RLDC/SLDC/utility) ownership | Dev-M | Migration 006 |
| 1.3.7 | Constraints and indexes, plus a test that loads IEEE 118 into the schema | Dev-M | Test passes in CI |

**WP1.4 Import and export (weeks 8 to 12)**

| # | Task | Owner | Done when |
|---|---|---|---|
| 1.4.1 | Converter: current DB -> CIM DB (uses `old_to_cim.csv`) | Dev-M | Real national model loads on a team machine |
| 1.4.2 | Importer: PSS/E .raw -> CIM DB (for cross-checks and other countries) | Dev-A | IEEE 118 .raw imports |
| 1.4.3 | CIM RDF/XML export and import (IEC 61970-552, EQ/TP/SSH/SV) | Dev-M | Round trip loses nothing on IEEE 118 |
| 1.4.4 | Bulk CSV/Excel import and export | Dev-M | Template + round trip test |

**WP1.5 Validation, topology and power flow check (weeks 10 to 14)**

| # | Task | Owner | Done when |
|---|---|---|---|
| 1.5.1 | Validation rules (missing terminals, islanded equipment, impossible parameters, voltage mismatches) | Dev-M | Report on national model |
| 1.5.2 | Topology processor: switch status -> buses/islands | Dev-A | Matches hand-checked cases |
| 1.5.3 | Newton-Raphson power flow (sparse, KLU) | Dev-A | IEEE 14/118/300 match published results; national model matches PSS/E within tolerance |
| 1.5.4 | "Publish model version" step that freezes a version for real-time use | Dev-M | Version id usable by RTDB |

Phase 1 demo: load the national model from the old DB into the CIM DB, validate it, run
topology and power flow, compare with PSS/E, export CIM XML.

### Phase 2: Real-time core (months 4 to 10)

- **WP2.1 RTDB:** shared-memory point store built from a model version; read/write API; quality codes; 4M-point load test.
- **WP2.2 FEP IEC 104:** in-house stack (APCI/ASDU, GI, clock sync, commands), redundant links; test against a simulator.
- **WP2.3 SCADA services:** limit checks, alarms (priority, acknowledge, shelve), events (SOE), calculations, control with select-before-operate, tagging.
- **WP2.4 Historian v1:** exception-based storage, deadband, compression, trend API, basic reports.
- **WP2.5 HMI v1:** web single-line diagrams from model (IEC 61970-453 layouts), alarm list, trends, RBAC login.
- **WP2.6 Replay tool:** feed recorded national SCADA data into the FEP at real speed or faster (main test method).

### Phase 3: Inter-centre and web (months 9 to 14)

- **WP3.1 ICCP/TASE.2** server and client (blocks 1, 2, 5; bilateral tables).
- **WP3.2 IEC 101** and PMU (C37.118) inputs.
- **WP3.3 Data replica + Web server** (apps 7 and 8): one-way replication to IT zone, web dashboards.
- **WP3.4 PDS & test bench** (app 6): integration test harness for FEP, SCADA and ICCP.
- **WP3.5 Load Shed Support / ADMS** (app 2).

### Phase 4: Core EMS network applications (months 6 to 20)

In dependency order: State Estimator (WLS, bad data detection, observability) ->
Real-time Contingency Analysis -> Security Enhancement -> Optimal Power Flow ->
Short Circuit (IEC 60909) -> Transmission Capability Monitor.
Each app: algorithm note -> C++ implementation -> IEEE cases -> national model vs PSS/E -> HMI display.

### Phase 5: Operational EMS and advanced apps (months 14 to 30)

Load Forecasting -> AGC (ACE, LFC, economic allocation) -> Outage Scheduler ->
DSA (VSA, TSA, SSA, needs dynamic models in CIM DY profile) -> DTS (reuses RTDB, PF,
dynamic simulation and a copy of the HMI).

### Cross-cutting (all phases)

- High availability: main/standby servers per function, main/backup control centre sync.
- Security: RBAC, MFA, audit log, encrypted storage, IEC 62351 on links, SBOM in CI, secure update, OT/IT separation.
- Performance tests against EOI sizes at every milestone.

---

## 6. Schema design plan (based on the current SCADA.SKM and OPENNET.SKM)

### 6.1 What the current database contains

| File | Tables | Fields | Notes |
|---|---|---|---|
| SCADA.SKM (OpenSCADA 3.4) | 53 | ~1,010 | Points (STATUS, ANALOG, ACCUMULATOR, SETPOINT), STATION, SCALE, LIMIT_GROUP, AOR_GROUP, 3-tier hierarchy, SOURCE_SELECTION, tags, quality |
| OPENNET.SKM (network model 4.12) | 101 | ~2,920 | COMPANY, AREA, REGION, NET_STATION, SBUS, BREAKER, BRANCH, XFORMER, TAP, GENER, LOAD, SHUNT, HVDC (DCTERM, DCLINE, T_DCTERM), SERIES_COMP, SERIES_FACTS, limits, curves, plus EMS app tables (CA, OPF, RAS, SE results) |

The full field list is in `current_schema_inventory.csv` (3,932 rows), with empty
`decision` and `cim_target` columns ready for WP1.1.

About 60% of the network model fields are inputs (data we enter). The rest are
application results (`{r}`), which are not part of the model database. They belong
to each EMS app's own output tables.

### 6.2 Findings that shape the new schema

1. **The CIM standard in the repository is too old.** `IEC 61970-301-2013.pdf` declares
   `IEC61970CIM14v15`. CIM14 has no PowerTransformerEnd, no solar, battery or
   power-electronics classes, and no CIM HVDC converter model. **Action:** get IEC 61970-301
   Ed.7 (CIM17) and the CGMES 3.0 profiles (IEC 61970-600-1/-2), and design on CIM17.
2. **Measurement links are text keys spread across equipment.** Each BREAKER row has 7
   SCADA keys (`pSCADA`, `pSCADA_mw`, `pSCADA_mvar_to`...), and SBUS, BRANCH, GENER and
   LOAD have their own. In CIM a Measurement object points at a Terminal, so one
   Measurement table replaces all of these.
3. **Two separate station hierarchies.** SCADA uses STATION and HIERARCHY_TIER1..3, while
   the network model uses COMPANY, AREA, REGION and NET_STATION. The new design has one
   Substation per physical station, shared by SCADA and EMS, and SCADA keys are kept as
   alternate names.
4. **Topology is already node-breaker.** Equipment connects to (station, node) pairs and
   SBUS holds busbars. This maps directly to CIM ConnectivityNode, BusbarSection and
   Terminal. DBUS (computed bus) maps to TopologicalNode.
5. **Fixed array sizes** (e.g. ANALOG 14,000 and BRANCH 25,000 rows, 16 fixed limit slots)
   are replaced by unbounded tables to reach the EOI's 20 lakh points.
6. **Per-unit values.** R and X are stored in p.u., so the converter must convert to
   ohms and siemens using the base kV and 100 MVA (CGMES convention).
7. **Proprietary source.** This schema belongs to a vendor product. Per EOI clause 6.10,
   we use it only to map and migrate data. The new schema is designed from IEC 61970
   and must not copy the vendor's structure.

### 6.3 Mapping plan: old table to CIM target

| Old table(s) | CIM target (CIM17) | Notes |
|---|---|---|
| REGION, HIERARCHY_TIER1..3 | GeographicalRegion, SubGeographicalRegion | One hierarchy for SCADA and EMS |
| COMPANY | Organisation (extension) and equipment owner | Indian utilities and states |
| AREA (agc_area) | ControlArea, TieFlow | AGC and interchange |
| NET_STATION + SCADA STATION | Substation | Merged, one mRID |
| VOLTAGE_CLASS, SBUS.kv_class | BaseVoltage, VoltageLevel | |
| SBUS + node numbers | BusbarSection, ConnectivityNode, Terminal | |
| DBUS | TopologicalNode (computed, not edited) | |
| BREAKER (by type) | Breaker, Disconnector, LoadBreakSwitch, GroundDisconnector | GROUND table maps to GroundDisconnector |
| BRANCH (line type) | Line, ACLineSegment | |
| XFORMER + BRANCH + TAP | PowerTransformer, PowerTransformerEnd, RatioTapChanger, PhaseTapChanger(Tabular) | TAP tables map to TapChangerTablePoint |
| GENER | SynchronousMachine and GeneratingUnit (Thermal/Hydro/Nuclear) | `mode` = condenser maps to SynchronousMachine.type |
| GENER (wind/solar types) | PowerElectronicsConnection, PowerElectronicsWindUnit, PhotoVoltaicUnit | CIM17 only |
| BESS (new) | BatteryUnit | CIM17 only |
| LOAD, LOAD_CHAR | EnergyConsumer / ConformLoad / NonConformLoad, LoadResponseCharacteristic | |
| SHUNT | LinearShuntCompensator, StaticVarCompensator | `group` maps to SVC control |
| SERIES_COMP, SERIES_FACTS | SeriesCompensator (+ extension for TCSC/UPFC) | |
| DCTERM, T_DCTERM, DCLINE, DC_XFORMER | CsConverter / VsConverter, DCLineSegment, DCNode, DCTerminal | |
| MVAR_CAP_CURVE | ReactiveCapabilityCurve, CurveData | |
| IHR_CURVE | HeatInputCurve (GeneratingUnit cost) | |
| BRANCH limits, FLOW_LIMITS, COND_RATINGS, TEMPERATURE_RATINGS | OperationalLimitSet, OperationalLimitType, CurrentLimit, ApparentPowerLimit, ActivePowerLimit, VoltageLimit | 16 slots become named limit sets (normal/emergency/season) |
| CA_DEFINITION, CA_OUTAGES | Contingency, ContingencyEquipment | |
| ONET_RAS, RAS_* | RemedialActionScheme, Stage, StageTrigger (CIM17) | |
| STATUS | Discrete, ValueAliasSet (state names) | |
| ANALOG | Analog, AnalogLimitSet, AnalogLimit | |
| ACCUMULATOR, ACCUM_PERIOD | Accumulator, AccumulatorLimit | |
| SETPOINT, CONTROL | Command, SetPoint (Control classes) | |
| RTU address fields (not in these files) | RemoteUnit, RemoteSource, CommunicationLink | Need the FEP/RTU schema file |
| SCALE, UNITS | Extension (scaling) + UnitSymbol/UnitMultiplier | |

**Best features to keep as project extensions** (CIM has nothing equivalent):
SOURCE_SELECTION (priority-based redundant source selection), LIMIT_GROUP (4 limit sets,
temperature-dependent and piecewise-linear limits), rate-of-change and sudden-ROC alarms,
flatline detection, chatter filtering (EQUIP), AOR groups, archive groups, tag groups,
SE measurement sigma/weights, EQUIP_SCHED (scheduled outages linked to the outage scheduler),
nomograms and ATC paths/flowgates.

### 6.4 Schema design steps (fills WP1.1 to WP1.3)

1. Get CIM17 and CGMES 3.0 documents (see 6.2 point 1).
2. Go through the inventory CSV and mark each input field as keep-in-CIM, keep-as-extension
   or drop. About 1,840 input fields to review; the domain lead decides on the doubtful ones.
3. Get the missing schema files the SCADA DB refers to: ALARMS.SKM, SYSTEM.SKM, and the
   RTU/FEP communication schema (RTU addresses are not in SCADA.SKM). Also AGC, historian
   and ICCP schemas if they exist.
4. Agree on the design conventions:
   - One table per CIM class.
   - UUID mRID as the key.
   - Shared parent tables only where CIM points to an abstract class (IdentifiedObject,
     Equipment, EquipmentContainer).
   - CGMES units.
   - Extensions kept in a separate schema.
   - Ownership by jurisdiction, versioning and an audit log on every table.
5. Write the schema as numbered migrations in this order: governance (versions,
   jurisdictions, audit), core and containment, equipment, HVDC, limits, measurements and
   SCADA points, then SCADA extensions and EMS app inputs.
6. Prove the schema by loading IEEE 118 and one region of the national model, then the
   whole national model.

## 7. Milestones mapped to the EOI

| Month | Milestone | What we show |
|---|---|---|
| 4 | M0 internal | CIM DB with national model, validation, PF matches PSS/E |
| 12 | EOI Milestone-1 | CIM DB + PDS, SCADA/FEP/RTDB/Historian/HMI v1, PF/SE running on replayed data |
| 24 | EOI Milestone-2 | All 19 apps functional; integration and interoperability tests; security assessment |
| 30 to 36 | EOI Milestone-3 | Performance at full scale, 7-day availability test, VAPT/source audit, pilot |

---

## 8. Sizing notes that drive design

- **Points:** 20 lakh analog + 20 lakh status = 4 million points in the RTDB. At ~64 bytes per point
  that is ~256 MB, which fits in memory on one server. Updates must be lock-free or sharded.
- **Historian:** 4 million points every 5 s for 10 years is about 2.5 x 10^14 samples, roughly
  2.5 PB uncompressed. Storing every sample is not realistic. The design must use
  report-by-exception, deadbands and compression (target 20x to 100x), and this needs to be
  agreed with the EOI working group as the interpretation of "5 s periodicity".
- **EMS model:** 10,000 buses, 25,000 lines. Sparse NR power flow should solve in well under 1 s;
  contingency analysis on thousands of outages needs parallel workers.

---

## 9. Immediate next steps (this week)

1. Team reviews this plan and the tech stack table; mark changes.
2. Get CIM17 / CGMES 3.0 documents and the missing schema files (section 6.4).
3. Phase 0 tasks 0.1 to 0.5 start in parallel.
4. Next document: `schema/cim/README.md` with the first migration (001 core tables), made from the
   current schema once it is available.
