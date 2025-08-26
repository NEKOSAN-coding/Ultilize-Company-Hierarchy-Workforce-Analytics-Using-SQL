# Ultilize-Company-Hierarchy-Workforce-Analytics-Using-SQL

## 1. 📝 Business Context
Prudential Vietnam Assurance (PVA) has recently acquired several subsidiaries through M&A activities. The management team requires a **standardized organizational data system** to:  
- Track the number of managers and employees in each company.  
- Evaluate managerial effectiveness (e.g., which Lead Manager has the most subordinates).  
- Establish a mechanism to select a **representative employee** for each company.  
- Improve **database architecture** for performance and scalability.  

---

## 2. 📂 Data Schema
The data structure reflects the organizational hierarchy:

```
Founder → Lead Manager → Senior Manager → Manager → Employee
```

**Tables:**
- `Company(company_code, founder)`  
- `Lead_Manager(lead_manager_code, company_code)`  
- `Senior_Manager(senior_manager_code, lead_manager_code, company_code)`  
- `Manager(manager_code, senior_manager_code, lead_manager_code, company_code)`  
- `Employee(employee_code, manager_code, senior_manager_code, lead_manager_code, company_code)`  

---

## 3. ❓ Business Questions
1. **Company Summary**: For each company, report the **founder** and the number of Lead Managers, Senior Managers, Managers, and Employees.  
2. **Managerial Effectiveness**: Which Lead Manager has the **largest number of subordinates** (including Senior Managers, Managers, and Employees)?  
3. **Representative Employee**: For each company, select one representative employee based on priority: smaller Lead Manager code → smaller Senior Manager code → smaller Manager code.  
4. **Data Architecture Review**: Evaluate the limitations of the current schema and propose improvements.  
5. **Roll-up Reporting**: For each Lead Manager, list all of their Senior Managers in a single cell (comma-separated).  

---

## 4. ⚙️ Approach & SQL Solutions

### Q1 – Company Summary
```sql
SELECT
    c.company_code,
    c.founder,
    COUNT(DISTINCT lm.lead_manager_code)   AS total_lead_manager,
    COUNT(DISTINCT sm.senior_manager_code) AS total_senior_manager,
    COUNT(DISTINCT m.manager_code)         AS total_manager,
    COUNT(DISTINCT e.employee_code)        AS total_employee
FROM Company AS c
LEFT JOIN Lead_Manager   lm ON lm.company_code = c.company_code
LEFT JOIN Senior_Manager sm ON sm.company_code = c.company_code
LEFT JOIN Manager        m  ON m.company_code  = c.company_code
LEFT JOIN Employee       e  ON e.company_code  = c.company_code
GROUP BY c.company_code, c.founder
ORDER BY TRY_CONVERT(INT, REPLACE(REPLACE(c.company_code,'C_',''),'C',''));
```

👉 **Business value**: Provides management with an overview of workforce distribution across companies after M&A.  

---

### Q2 – Lead Manager with Most Subordinates
```sql
WITH cnt AS (
    SELECT
        lm.lead_manager_code,
        COUNT(DISTINCT sm.senior_manager_code)
      + COUNT(DISTINCT m.manager_code)
      + COUNT(DISTINCT e.employee_code) AS total_subordinates
    FROM Lead_Manager AS lm
    LEFT JOIN Senior_Manager sm ON sm.lead_manager_code = lm.lead_manager_code
    LEFT JOIN Manager        m  ON m.lead_manager_code  = lm.lead_manager_code
    LEFT JOIN Employee       e  ON e.lead_manager_code  = lm.lead_manager_code
    GROUP BY lm.lead_manager_code
)
SELECT lead_manager_code, total_subordinates
FROM cnt
WHERE total_subordinates = (SELECT MAX(total_subordinates) FROM cnt);
```

👉 **Business value**: Identifies overloaded Lead Managers or those handling disproportionately large teams.  

---

### Q3 – Representative Employee per Company
```sql
WITH ranked AS (
    SELECT
        e.company_code,
        e.employee_code,
        ROW_NUMBER() OVER (
            PARTITION BY e.company_code
            ORDER BY e.lead_manager_code,
                     e.senior_manager_code,
                     e.manager_code,
                     e.employee_code
        ) AS rn
    FROM Employee e
)
SELECT company_code, employee_code
FROM ranked
WHERE rn = 1;
```

👉 **Business value**: Quickly designates a company’s representative employee to streamline communication.  

---

### Q4 – Schema Review & Proposal
- **Current issue**: Child tables store multiple parent keys (e.g., `employee` contains `manager_code`, `senior_manager_code`, `lead_manager_code`, `company_code`). → causes redundancy and risk of inconsistency.  
- **Proposed improvement**:
  - Store only the **immediate parent key** (Employee → Manager, Manager → Senior Manager, Senior Manager → Lead Manager, Lead Manager → Company).  
  - Add PK/FK and indexes to optimize joins.  
  - Add a `company_num INT` column for efficient numeric ordering.  
  - For deep hierarchy queries, consider **closure table** or **materialized path** approaches.  

👉 **Business value**: Easier to maintain, reduces data inconsistency risk, and improves query performance.  

---

### Q5 – Roll-up Senior Managers by Lead Manager
```sql
SELECT
    lm.lead_manager_code,
    STRING_AGG(sm.senior_manager_code, ',') WITHIN GROUP (ORDER BY sm.senior_manager_code) AS List_SENIOR_MANAGER
FROM Lead_Manager lm
LEFT JOIN Senior_Manager sm ON sm.lead_manager_code = lm.lead_manager_code
GROUP BY lm.lead_manager_code
ORDER BY lm.lead_manager_code;
```

👉 **Business value**: Provides clear roll-up reports for analyzing Lead Manager responsibilities.  

---

## 5. 📊 Key Business Insights
- Clear quantification of **workforce size by company**.  
- Easy identification of **Lead Managers with the largest span of control**.  
- Ability to **designate a single representative employee** for efficient reporting.  
- Schema redesign leads to **optimized storage, better performance, and reduced risk of data inconsistency**.  
