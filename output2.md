Good call — clean slate. Let me walk you through everything visually before you drop and restart.

---

## Step 1 — Drop & Recreate Schema

```sql
DROP DATABASE IF EXISTS ops_masterdb;
CREATE DATABASE ops_masterdb
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;
```

Then restart backend:
```powershell
cd "c:\Garage Pod\operation\VZ_GARAGE_OPS_EXCELENCE_POD\master-service-backend"
.\mvnw -DskipTests spring-boot:run
```

---

## Step 2 — Create These Two CSV Files

Save both files on your Desktop.

**File 1:** `test_people.csv`
```csv
name,role,age
raja,AD,23
ravi,VP,33
soumya,D,45
```

**File 2:** `test_reporting.csv`
```csv
name,rm_name
raja,soumya
ravi,soumya
```

---

## Step 3 — Full User Flow Visualization

---

### Flow A — Upload MASTER table (`test_people.csv`)

```
USER ACTION                    SYSTEM ACTION
────────────────────────────────────────────────────────────

1. Opens Upload page
   Enters:
   • Table Name:   test_people
   • Display Name: Test People

2. Selects type: MASTER         → No FK config needed

3. Drops test_people.csv        → Browser parses CSV headers
                                   Preview shows:
                                   ┌──────┬──────┬─────┐
                                   │ name │ role │ age │
                                   ├──────┼──────┼─────┤
                                   │ raja │ AD   │ 23  │
                                   │ ravi │ VP   │ 33  │
                                   └──────┴──────┴─────┘

4. Clicks Upload                → POST /api/master/upload
                                   • tableType = MASTER
                                   • file = test_people.csv
```

**What happens inside the backend:**
```
CsvIngestionService.ingest()
  ├── Parse CSV → 3 rows
  ├── For each row:
  │     Generate UUID → prepend as "test_people_sys_id"
  │     Keep all columns as-is
  │
  ├── Row 1: { test_people_sys_id: "uuid-aaa", name: "raja",   role: "AD", age: "23" }
  ├── Row 2: { test_people_sys_id: "uuid-bbb", name: "ravi",   role: "VP", age: "33" }
  └── Row 3: { test_people_sys_id: "uuid-ccc", name: "soumya", role: "D",  age: "45" }
```

**What gets stored in DB:**

`master_table_definitions` table:
```
id        | table_name   | display_name  | table_type | status | row_count | fk_mappings
──────────────────────────────────────────────────────────────────────────────────────────
uuid-def1 | test_people  | Test People   | MASTER     | LOCKED | 3         | null
```

`master_table_records` table:
```
id        | table_definition_id | row_data
──────────────────────────────────────────────────────────────────────────────────────────
uuid-r1   | uuid-def1           | {"test_people_sys_id":"uuid-aaa","name":"raja","role":"AD","age":"23"}
uuid-r2   | uuid-def1           | {"test_people_sys_id":"uuid-bbb","name":"ravi","role":"VP","age":"33"}
uuid-r3   | uuid-def1           | {"test_people_sys_id":"uuid-ccc","name":"soumya","role":"D","age":"45"}
```

**What user sees on success:**
```
┌─────────────────────────────────────┐
│ ✅ Upload Successful                 │
│                                     │
│ Table:   test_people                │
│ Type:    🗂 MASTER                   │
│ Rows:    3                          │
│ Status:  LOCKED                     │
│                                     │
│ Columns detected:                   │
│  • test_people_sys_id (auto-added)  │
│  • name                             │
│  • role                             │
│  • age                              │
└─────────────────────────────────────┘
```

---

### Flow B — Upload MAPPING table (`test_reporting.csv`)

```
USER ACTION                    SYSTEM ACTION
────────────────────────────────────────────────────────────

1. Opens Upload page
   Enters:
   • Table Name:   test_reporting
   • Display Name: Test Reporting

2. Selects type: REFERENCE      → FK mapping section appears

3. Drops test_reporting.csv     → Browser parses headers
                                   Preview shows:
                                   ┌──────┬─────────┐
                                   │ name │ rm_name │
                                   ├──────┼─────────┤
                                   │ raja │ soumya  │
                                   │ ravi │ soumya  │
                                   └──────┴─────────┘

4. Adds FK Mapping 1:
   CSV Column:        name        → dropdown shows: name, rm_name
   References Master: test_people → dropdown shows all MASTER tables
   Match on column:   name        → dropdown shows test_people columns

5. Adds FK Mapping 2:
   CSV Column:        rm_name
   References Master: test_people
   Match on column:   name

6. Clicks Upload                → POST /api/master/upload
                                   • tableType = REFERENCE
                                   • fkMappings = [
                                       { csvColumn: "name",
                                         referencedTable: "test_people",
                                         masterLookupColumn: "name" },
                                       { csvColumn: "rm_name",
                                         referencedTable: "test_people",
                                         masterLookupColumn: "name" }
                                     ]
```

**What happens inside the backend:**
```
CsvIngestionService.ingest()
  │
  ├── Parse CSV → 2 rows
  │
  ├── Build lookup map for test_people on column "name":
  │     "raja"   → "uuid-aaa"
  │     "ravi"   → "uuid-bbb"
  │     "soumya" → "uuid-ccc"
  │
  ├── Process Row 1: { name: "raja", rm_name: "soumya" }
  │     name    = "raja"   → lookup → "uuid-aaa"  ✅ resolved
  │     rm_name = "soumya" → lookup → "uuid-ccc"  ✅ resolved
  │     prepend mapping_sys_id = "uuid-m1"
  │     stored: { test_reporting_sys_id: "uuid-m1",
  │               name: "uuid-aaa",
  │               rm_name: "uuid-ccc" }
  │
  └── Process Row 2: { name: "ravi", rm_name: "soumya" }
        name    = "ravi"   → lookup → "uuid-bbb"  ✅ resolved
        rm_name = "soumya" → lookup → "uuid-ccc"  ✅ resolved
        stored: { test_reporting_sys_id: "uuid-m2",
                  name: "uuid-bbb",
                  rm_name: "uuid-ccc" }
```

**What gets stored in DB:**

`master_table_definitions` table:
```
id        | table_name      | table_type | status | row_count | fk_mappings
──────────────────────────────────────────────────────────────────────────────────────────────────────────
uuid-def2 | test_reporting  | REFERENCE  | LOCKED | 2         | [{"csvColumn":"name","referencedTable":
           |                 |            |        |           |  "test_people","masterLookupColumn":"name"},
           |                 |            |        |           |  {"csvColumn":"rm_name","referencedTable":
           |                 |            |        |           |  "test_people","masterLookupColumn":"name"}]
```

`master_table_records` table:
```
id        | table_definition_id | row_data
──────────────────────────────────────────────────────────────────────────────────────────────────────
uuid-rr1  | uuid-def2           | {"test_reporting_sys_id":"uuid-m1","name":"uuid-aaa","rm_name":"uuid-ccc"}
uuid-rr2  | uuid-def2           | {"test_reporting_sys_id":"uuid-m2","name":"uuid-bbb","rm_name":"uuid-ccc"}
```

---

### Flow C — View the mapping table in grid

```
USER ACTION                    SYSTEM ACTION
────────────────────────────────────────────────────────────

Clicks "Test Reporting" card   → GET /api/master/tables/
                                      test_reporting/records/resolved
                                      
                                  Backend:
                                  1. Fetch raw rows from DB
                                     row1: { name: "uuid-aaa", rm_name: "uuid-ccc" }
                                     row2: { name: "uuid-bbb", rm_name: "uuid-ccc" }
                                     
                                  2. Read fkMappings from definition
                                     name    → test_people → name column
                                     rm_name → test_people → name column
                                     
                                  3. Build reverse map from test_people:
                                     "uuid-aaa" → "raja"
                                     "uuid-bbb" → "ravi"
                                     "uuid-ccc" → "soumya"
                                     
                                  4. Substitute UUIDs → names:
                                     row1: { name: "raja",  rm_name: "soumya" }
                                     row2: { name: "ravi",  rm_name: "soumya" }
                                     
                                  5. Return resolved rows to frontend
```

**What user sees in grid:**
```
┌────────────────────────────────────────────────┐
│ Test Reporting                    🔗 REFERENCE  │
│                                                 │
│ References:                                     │
│  🔗 name     → Test People (matched on: name)   │
│  🔗 rm_name  → Test People (matched on: name)   │
├──────┬─────────┬────────┐                       │
│ name │ rm_name │        │                       │
├──────┼─────────┼────────┤                       │
│ raja │ soumya  │ ✏ Edit │                       │
│ ravi │ soumya  │ ✏ Edit │                       │
└──────┴─────────┴────────┘                       │
└────────────────────────────────────────────────┘
```

---

### Flow D — Edit a mapping row (ADMIN)

```
USER ACTION                    SYSTEM ACTION
────────────────────────────────────────────────────────────

Clicks Edit on row 1           → Modal opens
(raja → soumya)                   
                                  Backend fetches all test_people records:
                                  options = [
                                    { id: "uuid-aaa", display: "raja"   },
                                    { id: "uuid-bbb", display: "ravi"   },
                                    { id: "uuid-ccc", display: "soumya" }
                                  ]

Modal renders:                    
┌─────────────────────────────────┐
│ Edit Row                        │
│                                 │
│ name:                           │
│ [raja ▼]  ← current value      │
│  • raja   ← selected           │
│  • ravi                         │
│  • soumya                       │
│                                 │
│ rm_name:                        │
│ [soumya ▼] ← current value     │
│  • ravi    ← raja EXCLUDED ✅   │
│  • soumya                       │
└─────────────────────────────────┘

User changes rm_name → ravi     → Modal sends:
                                  PATCH /api/master/tables/
                                        test_reporting/records/{id}
                                  body: {
                                    rowData: {
                                      name:    "uuid-aaa",  ← raja's UUID
                                      rm_name: "uuid-bbb"   ← ravi's UUID
                                    }
                                  }

                                  → 202 Accepted
                                  → PendingChange created
```

---

### Flow E — Approve (SUPER_ADMIN)

```
Navbar → Approvals (badge: 1)

┌──────────────────────────────────────────────────┐
│ Pending Changes                                   │
│                                                   │
│ Table: test_reporting                             │
│ By:    bob.admin@example.com                      │
│ At:    2026-03-20 14:32                           │
├──────────────────────────────────────────────────┤
│ BEFORE              │ AFTER                       │
│─────────────────────────────────────────────────│
│ rm_name: soumya     │ rm_name: ravi               │
└──────────────────────────────────────────────────┘
         [Reject]  [Approve]
```

After Approve → grid shows:
```
name  | rm_name
raja  | ravi    ← updated ✅
ravi  | soumya
```

---

## Quick Verify After Backend Restart

Run this SQL after restart to confirm bootstrap ran:
```sql
SELECT table_name, table_type, row_count, status
FROM master_table_definitions
ORDER BY table_type, table_name;
```

Ready to test — drop the schema and restart. Paste the Spring Boot startup log here and we'll proceed. 🎯
