# PostgreSQL TOAST Internals 

1. **Page Layout Basics**:
   - Default page size is 8KB
   - TOAST tuple threshold is 2KB by default (aim to fit 4 records per page)
   - Problem: Some data types (text, JSON, etc.) can exceed page size

2. **TOAST Strategies**:
   - **Compression**: Try to compress large data to fit in-page
   - **TOAST Table**: Store data externally with pointer in main table
   - **Combination**: Both compression and external storage

3. **Storage Strategies**:

   | Strategy  | Compression | Toast Table | Example Data Types |
   |-----------|-------------|-------------|--------------------|
   | PLAIN     | No          | No          | integer, boolean   |
   | MAIN      | Yes         | Sometimes*  | numeric, inet      |
   | EXTENDED  | Yes         | Yes         | text, json, bytea  |
   | EXTERNAL  | No          | Yes         | (manually set)     |

   *Note: MAIN may use TOAST table if compression isn't sufficient

## Practical Demos

1. **PLAIN vs MAIN strategies**:
   - PLAIN tables (with integer/boolean) don't create TOAST tables
   - MAIN tables (with numeric/inet) create empty TOAST tables by default

2. **EXTENDED strategy**:
   - Text columns automatically create TOAST tables
   - Small text values (<2KB) stay in main table
   - Medium values may compress to fit in-page
   - Large values get sliced, compressed, and stored in TOAST table

3. **Performance Impact**:
   - Demonstrated with 2 tables (untoasted vs toasted)
   - Queries on toasted tables were 3-4x slower due to TOAST overhead
   - TOAST operations appear in query plans when they impact performance

## TOAST Internals

1. **Algorithm**:
   - Stage 1: Process longest EXTENDED/EXTERNAL fields first
   - Stage 2: If needed, compress MAIN fields
   - Stage 3: If still too large, move MAIN fields to TOAST table

2. **Configuration**:
   - `toast_tuple_threshold`: 2KB default (when TOAST kicks in)
   - `toast_max_chunk_size`: Controls chunk size in TOAST table

## Practical Tips

1. **Monitoring TOAST**:
   - Check if tables have TOAST tables
   - Identify columns' storage strategies
   - Find which tables in schema use TOAST
   - Check if TOAST tables are being scanned

2. **Compression Options**:
   - Default compression (slower)
   - LZ4 compression (newer, faster - benchmark for your use case)

3. **Performance Considerations**:
   - TOAST operations add overhead
   - For performance-critical apps, consider:
     - Using EXTERNAL strategy (no compression)
     - Using LZ4 compression
     - Avoiding unnecessary TOAST operations

