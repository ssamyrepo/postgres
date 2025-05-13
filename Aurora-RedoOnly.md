### **Aurora Writes Only Redo Log Records, Reducing Data Page Writes**
Amazon **Aurora's storage architecture** is fundamentally different from traditional **PostgreSQL on RDS**. The key difference is **how it handles writes**.

---

## **Traditional PostgreSQL (RDS) Write Process**
1. When you **INSERT/UPDATE/DELETE** data:
   - PostgreSQL writes changes to **both**:
     - **Write-Ahead Log (WAL)** (for durability and crash recovery).
     - **Data Pages** in the **data directory** (eventually written to disk).
   - Changes in **shared_buffers** (PostgreSQL memory cache) are periodically flushed to disk during:
     - **Checkpoints**
     - **Background writer process**
2. **Checkpoints** force all dirty pages (modified but not written) to be **flushed to disk**.
   - This process causes **high I/O spikes** and **increases disk latency**.

### **Downside of Traditional PostgreSQL:**
- **Two writes for every change** (one to WAL and another to data pages).
- **Checkpoints cause I/O spikes**.
- **Frequent disk writes increase storage costs and latency**.

---

## **Aurora's Storage Model: Redo Log-Only Writes**
Instead of writing **both WAL and data pages**, **Aurora writes only a "redo log"** and reconstructs data pages dynamically when needed.

### **How Aurora Writes Work:**
1. **User Transaction**
   - A client performs an **INSERT/UPDATE/DELETE**.
   - Instead of modifying the database page and writing it back to disk, Aurora **only records a redo log entry**.
   
2. **Aurora Log Record (Redo Log)**
   - The redo log **contains the change information**, not the actual data page.
   - Aurora stores **multiple copies** (across 6 storage nodes in 3 Availability Zones).
   - Aurora commits the change once **4 out of 6 copies confirm the write**.
   
3. **No Immediate Data Page Writes**
   - Since **only the redo log** is written, Aurora **avoids frequent page writes**.
   - Pages are **rebuilt on-demand from the redo log** when needed for queries.

4. **Read Operations and Page Construction**
   - If a query requests data that is not in memory, Aurora **fetches the latest version by replaying the redo log**.
   - This reduces storage I/O overhead significantly.

---

### **Why This Matters**
- **Fewer writes** → Lower I/O latency and **reduced disk I/O**.
- **No traditional checkpoints** → Avoids the **I/O spikes** seen in RDS PostgreSQL.
- **Better scaling** → Aurora can handle **high-throughput workloads** efficiently.

---

### **Trade-Offs of Aurora's Approach**
| Feature | Traditional PostgreSQL (RDS) | Aurora |
|---------|-----------------------------|--------|
| **Write Process** | Writes to both WAL and data pages | Writes only to **redo log** |
| **Checkpoints** | Required for crash recovery | No checkpoints needed |
| **Data Page Writes** | Frequent due to dirty page flushing | Rebuilt on demand using redo logs |
| **I/O Costs** | Higher due to **extra data page writes** | Lower because **only redo logs are written** |
| **Read Latency** | Lower for frequently accessed data | Can be slower if redo logs need to be replayed |

---

### **Best Practices for Aurora**
- **Turn off synchronous commit for non-critical writes** to **reduce write latency**.
- **Optimize queries** to reduce **redo log replay overhead**.
- **Monitor I/O costs** since Aurora charges based on **log writes and data reads**.

---

### **Summary**
- **Aurora avoids writing full data pages** by **storing only redo logs**.
- This **reduces write amplification** and **lowers storage I/O costs**.
- It **eliminates checkpoints**, preventing **I/O spikes**.
- **However, reads may require reconstructing pages**, impacting query latency if not optimized.

Great question — and a common one. Here's a concise and clear explanation of **why read operations in Amazon Aurora PostgreSQL can be slower** than in traditional PostgreSQL or PostgreSQL on RDS:

---

### 🔍 **Why Aurora Reads Can Be Slower**

| **Reason**                                 | **Explanation**                                                                                                                                                                                                      |
| ------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Distributed Storage Architecture**       | Aurora stores data on a distributed, log-structured storage system across 3 AZs. Even simple reads **may go over the network**, adding **latency** vs local disk or memory-based reads.                              |
| **No Local Data Files on Instances**       | In traditional PostgreSQL or RDS, data is stored **locally** on the instance's disk (e.g., EBS), which has lower latency for cached or nearby data. Aurora reads **always come from shared storage**, unless cached. |
| **Shared Buffer Efficiency**               | Although Aurora supports `shared_buffers`, **the cache hit ratio can be lower** than on RDS/PostgreSQL, especially on busy multi-tenant workloads or when instance memory is smaller.                                |
| **Tiered Cache Needs Tuning**              | Newer features like **Tiered Cache** and **Optimized Reads** (with NVMe) improve performance, but they’re only available on **specific instance families (R6id, R6gd)** and require correct configuration.           |
| **Read-Replica Lag (Asynchronous)**        | Aurora replicas sync changes **asynchronously** from the writer. Some read queries might hit **stale data** or wait on block invalidations if the reader buffer is behind.                                           |
| **CPU Contention on Multi-Tenant Storage** | Aurora’s back-end storage is multi-tenant. Under high load, or noisy-neighbor scenarios, read latencies may increase due to contention.                                                                              |

---

### 📊 Performance Example

| Workload Type        | Aurora PostgreSQL                                     | PostgreSQL RDS     | Self-Managed PostgreSQL    |
| -------------------- | ----------------------------------------------------- | ------------------ | -------------------------- |
| **Sequential Reads** | ⚠️ Slightly Slower (network read from shared storage) | ✅ Fast (local EBS) | ✅ Fastest (local SSD/NVMe) |
| **Index Scans**      | ⚠️ Slower if index not fully cached                   | ✅ Good             | ✅ Excellent                |
| **Buffer Cache Hit** | ⚠️ Depends on instance size & activity                | ✅ Tunable          | ✅ Fully tunable            |

---

### ✅ When Aurora Excels

* **Write-heavy workloads** (due to quorum log writes)
* **High availability + failover** (due to shared storage)
* **Auto-scaling or serverless patterns**
* **Global database** and cross-region replication

---

### 🛠️ How to Improve Aurora Read Performance

1. Use **Optimized Reads** (NVMe-backed instances like `r6gd`, `r6id`).
2. Enable **Tiered Cache** for large working sets.
3. Increase `shared_buffers` and `work_mem` in parameter groups.
4. Use **Aurora Read Replicas** in **the same AZ** as the application.
5. Offload heavy queries to clones or analytics-specific replicas.

---

Would you like benchmark examples comparing read latencies between Aurora and RDS/PostgreSQL for different workload patterns (e.g., index scan vs full table scan)?
