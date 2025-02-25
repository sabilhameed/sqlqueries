# Monthly Closing Inventory

Function: Inventory
Database: SGME, SME

```sql
-- Existing CTEs and subqueries remain unchanged

InventoryCTE AS (
    SELECT 
        MonthEndDates.MonthEndDate,
        T0.ItemCode, 
        T1.ItemName, 
        T0.WhsCode,
        T2.ItmsGrpNam, 
        T1.U_AXECode,
        T1.U_AXE, 
        ISNULL(STK.BalQty,0) AS InStockQty,
        ISNULL(STK.Value,0) AS InStockValue,
        ISNULL(APRes.APResQty,0) + ISNULL(GRN.GRNQty,0) AS InTransitQty,
        ISNULL(APRes.APResVal,0) + ISNULL(GRN.GRNVal,0) AS InTransitValue,
        PP9001.Currency,
        PP9001.Price,
        ISNULL(T1.U_CountryOrg,0) AS CountryOfOrigin,
        ISNULL(T1.U_BarCode,0) AS Barcode,
        TXM.GRPODate,
        APTX.APResDate,
        Brand = (SELECT [Name] FROM [@BRAND] WHERE [Code] = T1.U_BrandCode),
        T1.U_ABCC AS ABCClassification,
        ProductLine = (SELECT DISTINCT([U_LnName]) 
                       FROM [@PRODCAT] 
                       WHERE [U_LnCode] = T1.U_PrdCatLnCode AND [U_SubLnCode] = T1.U_PrdCatSubLnCode AND [U_BrandCode] = T1.U_BrandCode),
        T1.U_SMEMatStatus AS MEMatStatus,  -- Added MEMatStatus
        T1.U_SEMatStatus AS SEMatStatus    -- Added SEMatStatus
    FROM MonthEndDates
    CROSS JOIN OITW T0 
    INNER JOIN OITM T1 ON T1.ItemCode = T0.ItemCode 
    INNER JOIN OITB T2 ON T2.ItmsGrpCod = T1.ItmsGrpCod
    -- Existing LEFT OUTER JOINs remain unchanged
)

-- The final SELECT statement
SELECT
    MonthEndDate,
    ItemCode,
    ItemName,
    WhsCode,
    ItmsGrpNam,
    SUM(InStockQty) AS MonthlyClosingStockQty,
    SUM(InStockValue) AS MonthlyClosingStockValue,
    SUM(InTransitQty) AS MonthlyInTransitQty,
    SUM(InTransitValue) AS MonthlyInTransitValue,
    MAX(ABCClassification) AS Pareto,
    MAX(Brand) AS Brand,
    MAX(U_AXE) AS AXE,
    MAX(MEMatStatus) AS MEMatStatus,  -- Added MEMatStatus
    MAX(SEMatStatus) AS SEMatStatus   -- Added SEMatStatus
FROM InventoryCTE
GROUP BY MonthEndDate, ItemCode, ItemName, WhsCode, ItmsGrpNam
ORDER BY MonthEndDate, ItemCode, ItemName, WhsCode, ItmsGrpNam
```

```sql
-- Common Table Expression to generate month-end dates for 2022
WITH MonthEndDates AS (
    SELECT CONVERT(DATE, '2019-01-31') AS MonthEndDate -- Start with January 2019
    UNION ALL
    SELECT EOMONTH(DATEADD(MONTH, 1, MonthEndDate)) FROM MonthEndDates#(lf)    
		WHERE DATEADD(DAY, 1, EOMONTH(MonthEndDate, 1)) <= GETDATE() -- Continue until September 2023
),

-- Main Query
InventoryCTE AS (
    SELECT 
        MonthEndDates.MonthEndDate,
        T0.ItemCode, 
        T1.ItemName, 
        T0.WhsCode,
        T2.ItmsGrpNam, 
        T1.U_AXECode,
        T1.U_AXE, 
        ISNULL(STK.BalQty,0) AS InStockQty,
        ISNULL(STK.Value,0) AS InStockValue,
        ISNULL(APRes.APResQty,0) + ISNULL(GRN.GRNQty,0) AS InTransitQty,
        ISNULL(APRes.APResVal,0) + ISNULL(GRN.GRNVal,0) AS InTransitValue,
        PP9001.Currency,
        PP9001.Price,
        ISNULL(T1.U_CountryOrg,0) AS CountryOfOrigin,
        ISNULL(T1.U_BarCode,0) AS Barcode,
        TXM.GRPODate,
        APTX.APResDate,
        Brand = (SELECT [Name] FROM [@BRAND] WHERE [Code] = T1.U_BrandCode),
        T1.U_ABCC AS ABCClassification,
        ProductLine = (SELECT DISTINCT([U_LnName]) 
                       FROM [@PRODCAT] 
                       WHERE [U_LnCode] = T1.U_PrdCatLnCode AND [U_SubLnCode] = T1.U_PrdCatSubLnCode AND [U_BrandCode] = T1.U_BrandCode),
		T1.U_SMEMatStatus AS MEMatStatus,   -- Added MEMatStatus
        T1.U_SEMatStatus AS SEMatStatus		-- Added SEMatStatus
    FROM MonthEndDates
    CROSS JOIN OITW T0 
    INNER JOIN OITM T1 ON T1.ItemCode = T0.ItemCode 
    INNER JOIN OITB T2 ON T2.ItmsGrpCod = T1.ItmsGrpCod
    LEFT OUTER JOIN
(
    SELECT 
        MonthEndDate,
        M0.ItemCode, 
        M0.Warehouse, 
        SUM(M0.InQty) - SUM(M0.OutQty) AS BalQty, 
        SUM(M0.TransValue) AS Value
    FROM OINM M0 
    INNER JOIN MonthEndDates ON 1=1
    WHERE M0.DocDate <= MonthEndDates.MonthEndDate
    GROUP BY MonthEndDates.MonthEndDate, M0.ItemCode, M0.Warehouse 
) STK ON STK.ItemCode = T0.ItemCode AND STK.Warehouse = T0.WhsCode AND STK.MonthEndDate = MonthEndDates.MonthEndDate
LEFT OUTER JOIN
(
    SELECT 
        MonthEndDate,
        A1.ItemCode, 
        A1.WhsCode, 
        ISNULL(SUM(A1.OpenQty),0) AS APResQty, 
        ISNULL(SUM((A1.LineTotal / A1.Quantity) * A1.OpenQty),0) AS APResVal
    FROM PCH1 A1 
    INNER JOIN OPCH A2 ON A2.DocEntry = A1.DocEntry
    INNER JOIN MonthEndDates ON 1=1
    WHERE
        A2.isIns ='Y' AND A1.LineStatus ='O' AND A2.DocDate <= MonthEndDates.MonthEndDate AND A2.CANCELED = 'N'
    GROUP BY MonthEndDates.MonthEndDate, A1.ItemCode, A1.WhsCode
) APRes ON APRes.ItemCode = T0.ItemCode AND APRes.WhsCode = T0.WhsCode AND APRes.MonthEndDate = MonthEndDates.MonthEndDate
LEFT OUTER JOIN
(
    SELECT
        MonthEndDate,
        A1.ItemCode,
        A1.WhsCode,
        ISNULL(SUM(A1.Quantity),0) AS GRNQty,
        ISNULL(SUM(A1.LineTotal),0) AS GRNVal
    FROM OPDN A0
    INNER JOIN PDN1 A1 ON A1.DocEntry = A0.DocEntry
    INNER JOIN OPCH A2 ON A2.DocEntry = A1.BaseEntry
    INNER JOIN MonthEndDates ON 1=1
    WHERE A0.DocDate > MonthEndDates.MonthEndDate AND A2.isIns = 'Y' AND A2.DocDate <= MonthEndDates.MonthEndDate AND A0.CANCELED = 'N'
    GROUP BY MonthEndDates.MonthEndDate, A1.ItemCode, A1.WhsCode
) GRN ON GRN.ItemCode = T0.ItemCode AND GRN.WhsCode = T0.WhsCode AND GRN.MonthEndDate = MonthEndDates.MonthEndDate
LEFT OUTER JOIN
(
    SELECT 
        MonthEndDate,
        ItemCode, 
        Price, 
        Currency 
    FROM ITM1 
    CROSS JOIN MonthEndDates
    WHERE PriceList = 30
) PP9001 ON T0.ItemCode = PP9001.ItemCode AND PP9001.MonthEndDate = MonthEndDates.MonthEndDate
LEFT OUTER JOIN
(
    SELECT 
        MonthEndDate,
        M0.ItemCode,
        (
            SELECT ISNULL(MAX(M2.DocDate),0)
            FROM PDN1 M1
            INNER JOIN OPDN M2 ON M2.DocEntry = M1.DocEntry
            WHERE M1.ItemCode = M0.ItemCode AND M2.DocDate <= MonthEndDates.MonthEndDate
            GROUP BY M1.ItemCode
        ) AS GRPODate
    FROM OITM M0
    CROSS JOIN MonthEndDates
) TXM ON TXM.ItemCode = T0.ItemCode AND TXM.MonthEndDate = MonthEndDates.MonthEndDate
LEFT OUTER JOIN
(
    SELECT 
        MonthEndDate,
        P0.ItemCode,
        (
            SELECT ISNULL(MAX(X2.DocDate),0)
            FROM PCH1 X1
            INNER JOIN OPCH X2 ON X2.DocEntry = X1.DocEntry
            WHERE X1.ItemCode = P0.ItemCode AND X2.DocDate <= MonthEndDates.MonthEndDate
            GROUP BY X1.ItemCode
        ) AS APResDate
    FROM OITM P0
    CROSS JOIN MonthEndDates
) APTX ON APTX.ItemCode = T0.ItemCode AND APTX.MonthEndDate = MonthEndDates.MonthEndDate
WHERE ItmsGrpNam = 'SELLABLE'
AND (ISNULL(STK.BalQty,0) + ISNULL(APRes.APResQty,0) + ISNULL(GRN.GRNQty,0)) > 0
)
SELECT
    MonthEndDate,
    ItemCode,
    ItemName,
    WhsCode,
    ItmsGrpNam,
    SUM(InStockQty) AS MonthlyClosingStockQty,
    SUM(InStockValue) AS MonthlyClosingStockValue,
    SUM(InTransitQty) AS MonthlyInTransitQty,
    SUM(InTransitValue) AS MonthlyInTransitValue,
	MAX(ABCClassification) AS Pareto,
	MAX(Brand) AS Brand,
	MAX(U_AXE) AS AXE,
	MAX(MEMatStatus) AS MEMatStatus,  -- Added MEMatStatus
    MAX(SEMatStatus) AS SEMatStatus   -- Added SEMatStatus
    FROM InventoryCTE
GROUP BY MonthEndDate, ItemCode, ItemName, WhsCode, ItmsGrpNam
ORDER BY MonthEndDate,ItmsGrpNam, ItemCode, WhsCode;
```