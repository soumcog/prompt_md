Yes — but let me first verify the backend boots cleanly with the new bootstrap runner, then confirm the frontend is wired correctly end to end.

**Check 1 — Backend startup logs**
```powershell
cd "c:\Garage Pod\operation\VZ_GARAGE_OPS_EXCELENCE_POD\master-service-backend"
.\mvnw -DskipTests spring-boot:run
```

Watch for these specific log lines:
```
[Bootstrap] Starting data bootstrap...
[Bootstrap] Ingesting MASTER → account_master ...
[Bootstrap] account_master: 10 rows ingested
[Bootstrap] Ingesting MASTER → tower_master ...
[Bootstrap] tower_master: 15 rows ingested
...
[Bootstrap] Ingesting REFERENCE → client_hierarchy ...
[Bootstrap] client_hierarchy: FK resolution - client_name: 329 resolved, 0 unresolved
[Bootstrap] Bootstrap complete. 10 tables loaded.
Started MasterServiceBackendApplication in X.XXX seconds
```

**Check 2 — Verify DB directly**
```sql
USE ops_masterdb;
SELECT table_name, table_type, status, row_count 
FROM master_table_definitions 
ORDER BY table_type, table_name;
```

Expected:
```
+-------------------------+------------+--------+-----------+
| table_name              | table_type | status | row_count |
+-------------------------+------------+--------+-----------+
| account_master          | MASTER     | LOCKED |        xx |
| client_stakeholder      | MASTER     | LOCKED |       329 |
| employee_master         | MASTER     | LOCKED |        xx |
| service_line_master     | MASTER     | LOCKED |        xx |
| tower_master            | MASTER     | LOCKED |        xx |
| allocation              | REFERENCE  | LOCKED |        xx |
| client_hierarchy        | REFERENCE  | LOCKED |       329 |
| project_master          | REFERENCE  | LOCKED |        xx |
| project_tower_mapping   | REFERENCE  | LOCKED |        xx |
| tower_stakeholder_map   | REFERENCE  | LOCKED |        xx |
+-------------------------+------------+--------+-----------+
```

**Check 3 — Frontend running**
```powershell
cd "c:\Garage Pod\operation\VZ_GARAGE_OPS_EXCELENCE_POD\master-service-frontend"
npm run dev
```

---

## Complete Frontend Test Flow

Once both are running, open **http://localhost:5173** and follow this exact sequence:

---

### 1. Login as SUPER_ADMIN
- Email: `superadmin@example.com`
- Password: `SuperAdmin@`
- Expected: redirected to **SuperAdmin Dashboard**

---

### 2. View auto-bootstrapped tables
- Click **Master Tables** in navbar
- Expected: 10 table cards, split into two groups:

```
MASTER (5 cards)
├── Account Master
├── Client Stakeholder
├── Employee Master
├── Service Line Master
└── Tower Master

REFERENCE (5 cards)
├── Allocation
├── Client Hierarchy
├── Project Master
├── Project Tower Mapping
└── Tower Stakeholder Map
```

- Filter tabs at top: **All | Master | Reference**
- Each card shows: row count, column count, LOCKED badge, type badge

---

### 3. View a MASTER table
- Click **Client Stakeholder** card
- Expected:
  - Table with columns: `client_stakeholder_sys_id`, `Client Name`, `VZ Grade`, `VZ Role`, `Status`
  - `client_stakeholder_sys_id` is a UUID auto-generated per row
  - Pagination at bottom
  - Edit button on each row (visible to ADMIN + SUPER_ADMIN)

---

### 4. View a REFERENCE table
- Click **Client Hierarchy** card
- Expected:
  - FK panel at top showing:
    ```
    References:
    client_name → client_stakeholder (matched on: Client Name)
    reporting_manager → client_stakeholder (matched on: Client Name)
    ```
  - Columns include: `client_name`, `client_name_id` (UUID), `reporting_manager`, `reporting_manager_id` (UUID)
  - Rows where reporting manager was `NULL` → `reporting_manager_id` will be null

---

### 5. Upload a NEW table manually (test the wizard)
- Click **Upload CSV** in navbar (SUPER_ADMIN only)
- **Step 1 — Identity:**
  - Table Name: `test_master_table`
  - Display Name: `Test Master Table`
  - Click Next
- **Step 2 — Type:**
  - Select **MASTER**
  - Click Next
- **Step 3 — File:**
  - Drop any of the master CSVs (e.g. Account Master)
  - Verify preview shows first 3 rows + column headers detected
  - Click Next
- **Step 4 — Review & Upload:**
  - Click Upload
  - Expected success card:
    ```
    Table: test_master_table
    Type: MASTER
    Rows ingested: XX
    Columns: [test_master_table_sys_id, ...]
    Status: LOCKED
    ```

---

### 6. Upload a REFERENCE table manually
- Click **Upload CSV**
- **Step 1:** Name: `test_reference`, Display: `Test Reference`
- **Step 2:** Select **REFERENCE**
- **Step 3:** Drop `DB Inputs(Client Stakeholder (-1)).csv`
- **Step 4 — FK Mapping:**
  - Click **+ Add FK Mapping**
  - CSV Column: `Client Name`
  - References Master: `client_stakeholder`
  - Match on column: `Client Name`
  - Click **Upload**
- Expected: success card with FK resolution summary:
  ```
  FK Resolution:
  client_name → client_stakeholder
    ✓ 329 resolved
    ✗ 0 unresolved
  ```

---

### 7. Create ADMIN + test edit flow
- Navbar → **Manage Users** → **Create User**
- Fill:
  ```
  Name: Bob Admin
  Email: bob.admin@example.com
  Password: AdminP@ss1
  Role: ADMIN
  ```
- Log out → Log in as `bob.admin@example.com`
- Go to **Master Tables** → click any table
- Click **Edit** on any row
- Change a field value → Submit
- Expected: `202 Accepted` — pending change created

---

### 8. Approve as SUPER_ADMIN
- Log out → Log in as `superadmin@example.com`
- Navbar → **Approvals** (amber badge with count)
- See pending change from Bob Admin
- Click row → diff panel:
  ```
  Before          After
  ──────────────────────
  Grade: VP   →   SVP
  ```
- Click **Approve**
- Go back to table → verify change applied

---

### Common issues + fixes

| Symptom | Fix |
|---|---|
| 10 tables not showing | Check Spring Boot console for `[Bootstrap]` errors — paste here |
| `_sys_id` column missing | Bootstrap may have used old CsvIngestionService — restart backend cleanly |
| FK panel not showing on REFERENCE table | Hard refresh browser `Ctrl+Shift+R` |
| Upload wizard stuck on Step 3 | Check browser console for JS errors — likely a CSV parse issue |
| Approval badge not showing | Log out and back in — token may have stale role |
| `400 Bad Request` on upload | Check backend console — likely FK mapping validation error |

---

Paste the Spring Boot startup log here and I'll confirm everything is wired before you start testing. 🎯
