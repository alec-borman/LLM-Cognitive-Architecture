# The Salesforce Architect’s "North Star" Cheat Sheet

**Role:** Technical Architect / System Architect
**Focus:** Decision Frameworks, Governance, Scalability, and Limits.
**Philosophy:** Scale for the 1 millionth record while building the 1st.

---

## 1. The Holy Grail: Order of Execution
*The single most common source of "unexplainable" bugs. Memorize this sequence.*

1.  **System Validation Rules** (Page layout limits, field lengths, types)
2.  **Before-Save Flows** (Fast Field Updates)
3.  **Before Triggers** (Apex)
4.  **Validation Rules** (Custom)
5.  **Duplicate Rules**
6.  **After-Save Triggers** (Apex)
7.  **Assignment Rules**
8.  **Auto-Response Rules**
9.  **Workflow Rules** (*Legacy - Deprecated*)
10. **After-Save Flows**
11. **Escalation Rules**
12. **Entitlement Rules**
13. **Roll-Up Summary Fields** (Parent record update)
14. **Commit**

> **The Crux Point:** Do not mix logic types. If you use Apex Triggers for an object, avoid mixing in Before-Save Flows for the same object unless strictly governed. Race conditions live here.

---

## 2. Integration Patterns (Decision Matrix)
*Choose the right pattern to avoid coupling and timeout issues.*

| Requirement | Pattern | Mechanism | Pros | Cons |
| :--- | :--- | :--- | :--- | :--- |
| **Real-time UI update** | Request & Reply | REST/SOAP Callout | Immediate feedback | Tightly coupled; blocks UI; brittle |
| **Fire and Forget** | Fire & Forget | Platform Events / CDC | Decoupled; scalable | No immediate confirmation to user |
| **Large Data Sync (In)** | Batch Data Synch | Bulk API 2.0 | Low API cost; handles millions | High latency (Async) |
| **UI Mashup (No Copy)** | Data Virtualization | Salesforce Connect (OData) | No storage cost; real-time | Performance relies on external system |

> **Adversary Check:** Never perform a "Request & Reply" callout inside a database trigger. It holds the database transaction open. Use `@future` or `Queueable` to break the context.

---

## 3. Large Data Volume (LDV) & Skew
*Where systems die silently. The 100k+ record danger zone.*

### Ownership Skew
* **Definition:** A single user owns >10,000 records of the same object.
* **Symptom:** Performance degradation during sharing recalculations.
* **Fix:** Round-robin assignment logic. Never use a generic "System Admin" owner for high-volume data.

### Lookup Skew
* **Definition:** >10,000 records point to a single parent record (e.g., a generic "Unassigned" Account).
* **Symptom:** `UNABLE_TO_LOCK_ROW` errors during parallel loads.
* **Fix:** Use a "catch-all" bucket strategy (Account A, Account B) or clear the lookup if unnecessary.

### Query Selectivity (The 10% Rule)
* **Threshold:** Standard SOQL degrades after 100k records.
* **Fix:** Ensure queries are **Selective**.
    * Indexed field in `WHERE` clause.
    * Returns <10% of total records (up to 1M).
    * Returns <5% of total records (over 1M).

---

## 4. Data Visibility (Sharing Architecture)
*The hierarchy of access. Top overrides Bottom.*

1.  **Profile/Permission Set:** "View All Data" (Absolute override)
2.  **Org-Wide Defaults (OWD):** The Baseline (Private, Public Read Only, Public Read/Write)
3.  **Role Hierarchy:** Vertical Access (Managers see what subordinates own)
4.  **Sharing Rules:** Horizontal Access (Criteria-based or Owner-based)
5.  **Manual Sharing:** Ad-hoc single record access
6.  **Apex Managed Sharing:** The "Nuclear Option" (Complex programmatic logic)

> **Blind Spot:** **Implicit Sharing.** Parent-Child relationships (like Account-Opportunity) grant implicit read/write access to parents for owners of children. This often fails security audits.

---

## 5. Asynchronous Apex Governance
*Processing Logic off the main thread.*

| Tool | Limit | Best For | Context |
| :--- | :--- | :--- | :--- |
| **Future Methods** | Primitive types only | Simple "fire & forget" | Moving callouts out of triggers |
| **Queueable** | Objects/Complex types | Chaining jobs | Robust background processing |
| **Batch Apex** | Up to 50M records | Large datasets | Nightly maintenance; heavy calculation |
| **Schedulable** | N/A | Timing | Kickstarting Batches |

**Critical Limits:**
* **Sync Heap Size:** 6 MB
* **Async Heap Size:** 12 MB
* **CPU Time:** 10,000 ms (10s) - *Shared across the transaction.*

---

## 6. Deployment & Lifecycle
* **Change Sets:** Native, manual, hard to track history.
* **Package Development Model (2GP):** Best practice. Source of truth is Version Control (Git). Modular, versioned, dependency-managed.
* **Sandbox Strategy:**
    * *Dev:* Individual config/code.
    * *Partial Copy:* UAT / QA (Sample data).
    * *Full Copy:* Staging / Load Testing / Training (Full data match).
