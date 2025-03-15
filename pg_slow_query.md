### **How to Compare Postgres Plans & Tune Slow Queries with pganalyze**


---

### **Key Takeaways**
1. **Identifying Slow Queries**
   - **pganalyze helps track queries** causing high **CPU/IO usage**.
   - Uses **pg_stat_statements** and **auto_explain** to log query behavior.
   - Detects **query outliers** that run significantly slower than usual.

2. **Plan Fingerprinting & Statistics**
   - New feature in **pganalyze** tracks **execution plan changes**.
   - Helps determine if a query plan has changed **without needing manual comparisons**.
   - Aurora users get **"Aurora stat plans"** integration for **plan tracking**.

3. **New Plan Comparison Feature**
   - **Compares two different query plans** side by side.
   - Highlights **structural differences** (e.g., **Index Scan vs. Sequential Scan**).
   - **Uses color-coded differences** to make analysis faster.

4. **Debugging Slow Queries**
   - PostgreSQL’s **planner sometimes picks inefficient plans**.
   - Uses **EXPLAIN ANALYZE** and **EXPLAIN (ANALYZE, BUFFERS)** for deeper insights.
   - Looks at **misestimated row counts** and **wrong join order** issues.

5. **Benchmarking Query Plans**
   - **Forcing alternative execution plans** using:
     - `SET enable_seqscan = OFF;` to test **Index Scans**.
     - `SET enable_nestloop = OFF;` to test **Hash Joins**.
   - Uses **PG Hint Plan** extension to **manually guide the planner**.

6. **Query Tuning Workbooks**
   - New feature in **pganalyze (Early Access)**.
   - Allows **iterative testing of different query plans**.
   - Stores **query execution results** for team collaboration.
   - Helps **validate queries against different input parameters**.

7. **Upcoming: Query Tuning Advisor**
   - Will **automatically detect problematic queries** and suggest fixes.
   - Covers **inefficient nested loops, wrong index usage, bad order by limit**, etc.
   - Designed to **reduce manual debugging efforts**.

---

### **Practical Tips for PostgreSQL Query Optimization**
✅ Use **pg_stat_statements** to find slow queries.  
✅ Run `EXPLAIN (ANALYZE, BUFFERS)` to inspect **execution details**.  
✅ **Compare different query plans** using pganalyze’s new **Plan Comparison Tool**.  
✅ **Turn off specific planner features (`SET enable_* = OFF`)** to test alternative plans.  
✅ Use **PG Hint Plan** to manually override PostgreSQL’s planner.  
✅ **Increase `random_page_cost` for cloud-based databases** to improve indexing.  
✅ Use **partitioning & multi-column indexes** for large datasets.  
✅ Consider **CTE materialization** (`MATERIALIZED` keyword) for better join performance.  

---

### **Conclusion**
- **pganalyze's new features** make **PostgreSQL query tuning much easier**.
- **Plan comparison, query tuning workbooks, and advisor** provide **practical debugging tools**.
- **Early Access** is available for **real-world database performance tuning**.

### **Summary of the Webinar: Optimizing Slow Queries with EXPLAIN to Fix Bad Query Plans**  

**Presenter:** Lucas, Founder of PG Analyze  

**Objective:**  
The webinar focuses on optimizing slow PostgreSQL queries using `EXPLAIN ANALYZE`. The goal is to help engineers diagnose inefficient query plans and improve query performance through better understanding of PostgreSQL’s query planner.

---

### **Key Takeaways**  

#### **1. General Process of Query Optimization**
- Start by identifying slow queries through monitoring tools like PG Analyze or PostgreSQL logs.
- Use `EXPLAIN ANALYZE` to get execution plans and runtime statistics.
- Identify potential improvements (e.g., rewriting queries, adding indexes, changing planner settings).
- Apply the change, benchmark the impact, and make the optimization permanent.

---

#### **2. Understanding `EXPLAIN ANALYZE` and Buffers**
- `EXPLAIN`: Shows the planner's estimated execution plan.
- `EXPLAIN ANALYZE`: Runs the query and provides actual execution times and row counts.
- `EXPLAIN ANALYZE BUFFERS`: Displays I/O statistics, revealing whether queries are slowed by disk access.
- **Key Metrics to Watch:**
  - Startup Cost: Effort to get the first row.
  - Total Cost: Estimated cost of full execution.
  - Buffers Hit: Number of pages fetched from memory (low is better).
  - Buffers Read: Number of pages read from disk (high can indicate a problem).
  - Rows Removed by Filter: Shows how much unnecessary data was processed.

---

#### **3. Common Causes of Bad Query Plans**
- **Inaccurate Planner Estimates:** PostgreSQL’s planner is not perfect and may misestimate row counts.
- **Bloating:** If a table has excessive dead tuples, queries become inefficient.
- **Join Order Issues:** PostgreSQL may pick a suboptimal order when joining multiple tables.
- **Inefficient Index Usage:** The planner sometimes picks a sequential scan when an index scan would be better.
- **Bounded Sorts Gone Wrong:** PostgreSQL may choose an inefficient index when sorting data.

---

#### **4. Techniques to Improve Query Performance**
✅ **Use `EXPLAIN ANALYZE` to Validate Query Performance:**  
   - Compare estimated vs. actual row counts.
   - Look for large mismatches to identify planner misestimates.  

✅ **Leverage Indexing Correctly:**  
   - Use multi-column indexes when filtering and sorting on multiple columns.  
   - Consider `BRIN` indexes for large, ordered datasets.  
   - Ensure indexes are selective enough to avoid unnecessary scans.  

✅ **Optimize Joins & Query Execution Order:**  
   - Use `EXPLAIN` to check if PostgreSQL is misordering joins.  
   - If the planner picks a bad join order, experiment with `SET enable_nestloop = off` to force a different execution plan.  

✅ **Understand Nested Loop vs. Hash Joins:**  
   - PostgreSQL chooses nested loops for small datasets but can be slow for large tables.  
   - Hash joins are often more efficient for large joins.  

✅ **Avoid Bloated Tables:**  
   - Run `VACUUM ANALYZE` regularly to update PostgreSQL statistics.  
   - Consider `pg_repack` for heavily bloated tables.  

✅ **Force Query Plan Testing with Hints & Settings:**  
   - Use **PG Hint Plan** extension to experiment with forcing PostgreSQL to use specific query plans.  
   - Adjust `random_page_cost` to influence index scan choices (e.g., lowering it can make index scans more attractive).  

---

#### **5. PostgreSQL Configuration Tuning**
- **Enable Planner Settings (if applicable):**  
   - `enable_partitionwise_join = on` → Helps with partitioned tables.  
   - `enable_partitionwise_aggregate = on` → Improves aggregation performance in partitioned tables.  
- **Adjust `random_page_cost` for Better Indexing Decisions:**  
   - Default is `4`, but lowering to `1.1` improves performance on modern SSD storage.  

---

### **Q&A Highlights**
- **How do you know if a query cannot be optimized further?**  
  - If returning large amounts of data, optimization is limited.  
  - Try simplifying the query and testing partial execution plans.  

- **How to speed up INSERT queries?**  
  - Use **batch inserts** (`COPY` instead of multiple `INSERT` statements).  
  - Avoid excessive indexes on frequently updated tables.  

- **Who to follow for PostgreSQL performance insights?**  
  - PostgreSQL mailing lists (`pgsql-performance`).  
  - Experts like Tom Lane and community blog posts.  

---

### **Conclusion**
- Query optimization is an **iterative process**—understand the planner’s decisions, experiment with indexing and configurations, and benchmark improvements.
- PostgreSQL’s planner isn’t always perfect—**monitor misestimates and bloating** to avoid unnecessary slow queries.
- Tools like **PG Analyze, Auto Explain, and PG Hint Plan** can help diagnose performance bottlenecks.

