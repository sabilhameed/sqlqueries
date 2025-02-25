# Last 6M Sell Out - KSA

Function: Sales Data
Database: SAR

```sql
SELECT 
    T1.[U_Country],           -- Select the country from T1 (detail table)
    T1.[U_ItemCode],          -- Select the item code from T1
    T1.[U_ItemName],          -- Select the item name from T1
    T3.[U_BarCode],           -- Select the barcode from T3 (items table)
    SUM(T1.[U_Qty]) AS Total_Qty,            -- Sum the quantities from T1 and alias it as Total_Qty
    T1.[U_Currency],          -- Select the currency from T1
    SUM(T1.[U_ValueUSD]) AS Total_ValueUSD,  -- Sum the values in USD from T1 and alias it as Total_ValueUSD
    T3.[U_BrandName]          -- Select the brand name from T3 (items table)
FROM [dbo].[@S_OSODC] T0      -- Main table T0
INNER JOIN [dbo].[@S_SODC11] T1 ON T1.[DocEntry] = T0.[DocEntry]  -- Join detail table T1 on DocEntry
LEFT OUTER JOIN OCRD T2 ON T2.[CardCode] = T1.[U_BPCode]          -- Left join with business partners table T2 on BPCode
LEFT OUTER JOIN OITM T3 ON T3.[ItemCode] = T1.[U_ItemCode]        -- Left join with items table T3 on ItemCode
WHERE 
    T0.[U_Type] = 'O'         -- Filter for type 'O' in T0
    AND T1.[U_Qty] <> 0       -- Filter out rows where quantity is 0 in T1
    AND T1.[U_ValueUSD] <> 0  -- Filter out rows where value in USD is 0 in T1
    AND T1.[U_Date] >= DATEADD(MONTH, -7, CAST(GETDATE() AS DATE)) -- Filter for dates within the last 7 months
    AND T1.[U_Date] <= CONVERT(VARCHAR(8), GETDATE(), 112)        -- Filter for dates up to today
    AND T3.[U_BrandName] NOT IN ('DG', 'LM', 'ES')                -- Exclude specified brand names in T3
GROUP BY
    T1.[U_Country],           -- Group by country from T1
    T1.[U_ItemCode],          -- Group by item code from T1
    T1.[U_ItemName],          -- Group by item name from T1
    T3.[U_BarCode],           -- Group by barcode from T3
    T1.[U_Currency],          -- Group by currency from T1
    T3.[U_BrandName]          -- Group by brand name from T3
ORDER BY
    T3.[U_BrandName] ASC,     -- First sort by brand name alphabetically
    SUM(T1.[U_ValueUSD]) DESC -- Then sort by value within each brand in descending order

```

```sql
SELECT 
    T1.[U_Country],
    T1.[U_ItemCode],
    T1.[U_ItemName],
    T3.[U_BarCode],  -- Including the U_BarCode column from the OITM table
    SUM(T1.[U_Qty]) AS Total_Qty,
    T1.[U_Currency],
    SUM(T1.[U_ValueUSD]) AS Total_ValueUSD,
    T3.[U_BrandName]
FROM [dbo].[@S_OSODC] T0 
INNER JOIN [dbo].[@S_SODC11] T1 ON T1.[DocEntry] = T0.[DocEntry]
LEFT OUTER JOIN OCRD T2 ON T2.[CardCode] = T1.[U_BPCode]
LEFT OUTER JOIN OITM T3 ON T3.[ItemCode] = T1.[U_ItemCode]
WHERE 
    T0.[U_Type] = 'O' AND 
    T1.[U_Qty] <> 0 AND 
    T1.[U_ValueUSD] <> 0 AND 
    T1.[U_Date] >= DATEADD(MONTH, -7, CAST(GETDATE() AS DATE)) AND 
    T1.[U_Date] <= CONVERT(VARCHAR(8), GETDATE(), 112) AND 
    T3.[U_BrandName] NOT IN ('DG', 'LM', 'ES') AND
    T2.[U_Country] = 'SA'
GROUP BY
    T1.[U_Country],
    T1.[U_ItemCode],
    T1.[U_ItemName],
    T3.[U_BarCode],  -- Group by U_BarCode as well
    T1.[U_Currency],
    T3.[U_BrandName]
ORDER BY
    T3.[U_BrandName] ASC,  -- First sort by brand name alphabetically
    SUM(T1.[U_ValueUSD]) DESC  -- Then sort by value within each brand in descending order

```

### Explanation:

1. **SELECT Clause**:
    - Columns are selected from table `T1` (detail table) and table `T3` (items table).
    - Aggregation functions (`SUM`) are used for `U_Qty` and `U_ValueUSD`.
2. **FROM Clause**:
    - The main table `T0` is joined with detail table `T1` using an inner join on `DocEntry`.
    - Left joins are performed with tables `T2` (business partners) and `T3` (items).
3. **WHERE Clause**:
    - Various filters are applied:
        - Type 'O' in `T0`
        - Non-zero quantity and value in USD in `T1`
        - Dates within the last 7 months in `T1`
        - Specific brand names are excluded from `T3`
        - Country 'SA' in `T2`
4. **GROUP BY Clause**:
    - Grouping is done on columns from `T1` and `T3`.
5. **ORDER BY Clause**:
    - The result is ordered by the sum of values in USD in descending order and by brand name in ascending order.

**Suggestions for further improvements:a.** Add indexes on the columns used in joins and filters to improve query performance.
**b.** Validate the data types of the columns used in joins and conditions to ensure compatibility and avoid implicit conversions.