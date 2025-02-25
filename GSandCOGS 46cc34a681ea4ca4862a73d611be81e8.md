# GSandCOGS

Function: Sales Data
Database: SGME, SME

```sql
-- Use the ShiseidoME_Live database
USE ShiseidoME_Live;

-- Declare start and end date variables for filtering
DECLARE @StartDate DATETIME;
DECLARE @EndDate DATETIME;

-- Set the date variables
SET @StartDate = CONVERT(DATETIME, '2019-01-01 00:00:00.000', 121);
SET @EndDate = CAST(GETDATE() AS DATETIME);

-- Fetch Invoice Data
SELECT 
    'IN' AS "DocType",  -- Hardcoded to identify as Invoice
    T0."CANCELED", 
    T0."DocNum", 
    YEAR(T0."DocDate") AS "Year", 
    DATENAME(MM, T0."DocDate") AS "Month", 
    T0."CardCode", 
    T0."CardName", 
    T1."OcrCode" AS "Country", 
    T1."ItemCode", 
    T1."Dscription",
    T1."OcrCode2" AS "Brand", 
    -- Handling Quantity based on Cancellation status
    CASE WHEN T0."CANCELED" <> 'C' THEN T1."Quantity" ELSE T1."Quantity" * -1 END AS "Quantity", 
    -- Handling Sales Value based on Cancellation status
    CASE WHEN T0."CANCELED" <> 'C' THEN T1."LineTotal" ELSE T1."LineTotal" * -1 END AS "Sales Value", 
    T1."AcctCode" AS "Revenue Account", 
    -- Handling COGS based on Cancellation status
    CASE WHEN T0."CANCELED" <> 'C' THEN T1."StockValue" ELSE T1."StockValue" * -1 END AS "COGS Actual",  
    T1."CogsAcct" AS "COGS Account",
    -- Calculating Price Differential
    (T1."StockValue" + ISNULL(BStk."COGS_SC", 0)) AS "Price Differential", 
    ISNULL(BStk."COGS_SC", 0) * -1 AS "COGS SC",
    -- Calculating Landed Cost
    (ISNULL(BStk."COGS_SC", 0) * -1) - 
    (ISNULL((SELECT "Price" FROM ITM1 WHERE "ItemCode" = T1."ItemCode" AND "PriceList" = 70), 0) * ISNULL(T1."Quantity", 0)) AS "Landed Cost",
    -- Fetching CreatedBy user
    (SELECT "U_Name" FROM OUSR WHERE "Internal_K" = T0."UserSign") AS "CreatedBy", 
    T2.U_BrandName
FROM 
    OINV T0 
    INNER JOIN INV1 T1 ON T1."DocEntry" = T0."DocEntry"
    INNER JOIN OITM T2 ON T2."ItemCode" = T1."ItemCode"
    LEFT JOIN (
        -- Subquery to fetch Batch Stock Data
        SELECT 
            "DocEntry", 
            "Line_No", 
            SUM("COGS_At_SC") AS "COGS_SC"
        FROM 
            SBOS_BATCH_STOCK
        WHERE 
            "TransType" = '15' AND "Cost_At_Actual" <> 0
        GROUP BY 
            "DocEntry", "Line_No"
    ) BStk ON BStk."DocEntry" = T1."BaseEntry" AND BStk."Line_No" = T1."BaseLine"
WHERE 
    T0."CANCELED" = 'N' AND T2."ItmsGrpCod" = 100 AND T0."DocType" = 'I' 
    AND T1."AcctCode" IN ('70150010', '70150011') AND T1."CogsAcct" = '60150000' AND T0."CardCode" <> 'AEC1042-02'
    AND CAST(T0."DocDate" AS DATE) BETWEEN @StartDate AND @EndDate

-- Union to combine Invoice and Credit Note Data
UNION ALL

-- Fetch Credit Note Data (similar logic as Invoice but with different tables and conditions)
SELECT 
    'CN' AS "DocType", 
    T0."CANCELED", 
    T0."DocNum",  
    YEAR(T0."DocDate") AS "Year", 
    DATENAME(MM, T0."DocDate") AS "Month", 
    T0."CardCode", 
    T0."CardName", 
    T1."OcrCode" AS "Country", 
    T1."ItemCode", 
    T1."Dscription",
    T1."OcrCode2" AS "Brand", 
    CASE 
        WHEN T0."CANCELED" <> 'C' THEN T1."Quantity" * -1 
        ELSE T1."Quantity"
    END AS "Quantity",
    CASE 
        WHEN T0."CANCELED" <> 'C' THEN 
            CASE 
                WHEN T1."AcctCode" IN ('70150010', '70150011') THEN T1."LineTotal" * -1 
                ELSE 0 
            END
        ELSE 
            CASE 
                WHEN T1."AcctCode" IN ('70150010', '70150011') THEN T1."LineTotal" 
            END
    END AS "Sales Value",
    T1."AcctCode" AS "Revenue Account",
    CASE 
        WHEN T0."CANCELED" <> 'C' THEN T1."StockValue" * -1 
        ELSE T1."StockValue"
    END AS "COGS Actual",
    T1."CogsAcct" AS "COGS Account",
    (T1."StockValue" - ISNULL(BStk."COGS_SC", 0)) * -1 AS "Price Differential",
    ISNULL(BStk."COGS_SC", 0) * -1 AS "COGS SC",
    (ISNULL(BStk."COGS_SC", 0) * -1) - 
    (ISNULL((SELECT "Price" FROM ITM1 WHERE "ItemCode" = T1."ItemCode" AND "PriceList" = 70), 0) * ISNULL(T1."Quantity", 0)) AS "Landed Cost",
    (SELECT "U_Name" FROM OUSR WHERE "Internal_K" = T0."UserSign") AS "CreatedBy", 
    T2.U_BrandName
FROM 
    ORIN T0 
    INNER JOIN RIN1 T1 ON T1."DocEntry" = T0."DocEntry"
    INNER JOIN OITM T2 ON T2."ItemCode" = T1."ItemCode"
    LEFT JOIN (
        SELECT 
            "DocEntry", 
            "Line_No", 
            SUM("COGS_At_SC") AS "COGS_SC"
        FROM 
            SBOS_BATCH_STOCK
        WHERE 
            "TransType" = '14' AND "Cost_At_Actual" <> 0
        GROUP BY 
            "DocEntry", "Line_No"
    ) BStk ON BStk."DocEntry" = T1."DocEntry" AND BStk."Line_No" = T1."LineNum"
WHERE 
    T0."CANCELED" = 'N' AND T2."ItmsGrpCod" = 100 AND T0."DocType" = 'I' 
    AND T1."CogsAcct" = '60150000' AND T0."CardCode" <> 'AEC1042-02'
    AND CAST(T0."DocDate" AS DATE) BETWEEN @StartDate AND @EndDate;
```

```sql
-- Declare variables for date range
DECLARE @StartDate DATE = '2023-01-01';
DECLARE @EndDate DATE = CAST(GETDATE() AS DATE);

-- First part of the UNION (OINV)
Select T0.TaxDate, (convert(char(3), T0.[TaxDate], 0)) 'Month', CONVERT(char(4), YEAR(T0.[DocDate])) 'Year', 
T0.CardCode 'Customer Code', T0.CardName 'Customer Name', T6.GroupName 'Customer Group', 
ISNULL(T2.FrgnName,'') 'Global Code', T1.ItemCode, T2.ItemName, ISNULL(T3.Name,'') 'AXE', ISNULL(T2.U_Dim4PrdLine,'') 'Product Line', ISNULL(T2.U_PrdCatLnName,'') 'Product Category', ISNULL(T2.U_PrdCatSubLnName,'') 'Product Sub Category',
T1.Quantity, T1.Currency, T1.Price, T1.LineTotal 'Total LC', T1.TotalFrgn 'Total FC', T1.WhsCode, T9.WhsName,
ISNULL(T1.CogsOcrCod,'') 'COGS Country', ISNULL(T4.CountryB,'') 'Country Code', ISNULL(T8.Name,'') 'Country Name', T1.BaseCard 'Store', T7.CardName 'Store Name' 
from OINV T0 
Inner Join INV1 T1 On T1.DocEntry = T0.DocEntry
Inner Join OITM T2 On T2.ItemCode = T1.ItemCode 
Inner Join [@AXE] T3 On T3.Code = T2.U_AXECode
Inner Join INV12 T4 On T4.DocEntry = T0.DocEntry
Inner Join OCRD T5 On T5.CardCode = T0.CardCode
Inner Join OCRG T6 On T6.GroupCode = T5.GroupCode 
Left Outer Join OCRD T7 On T7.CardCode = T1.BaseCard
Left Outer Join OCRY T8 On T8.Code = T4.CountryB
Inner Join OWHS T9 On T9.WhsCode = T1.WhsCode
Where T0.CANCELED ='N' and T0.DocType ='I' and T1.Price<>0
and T0.TaxDate between @StartDate and @EndDate

UNION ALL

-- Second part of the UNION (ORIN)
Select T0.TaxDate, (convert(char(3), T0.[TaxDate], 0)) 'Month', CONVERT(char(4), YEAR(T0.[DocDate])) 'Year', 
T0.CardCode 'Customer Code', T0.CardName 'Customer Name', T6.GroupName 'Customer Group', 
ISNULL(T2.FrgnName,'') 'Global Code', T1.ItemCode, T2.ItemName, ISNULL(T3.Name,'') 'AXE', ISNULL(T2.U_Dim4PrdLine,'') 'Product Line', ISNULL(T2.U_PrdCatLnName,'') 'Product Category', ISNULL(T2.U_PrdCatSubLnName,'') 'Product Sub Category',
-T1.Quantity, T1.Currency, T1.Price, -T1.LineTotal 'Total LC', -T1.TotalFrgn 'Total FC', T1.WhsCode, T9.WhsName,
ISNULL(T1.CogsOcrCod,'') 'COGS Country', ISNULL(T4.CountryB,'') 'Country Code', ISNULL(T8.Name,'') 'Country Name', T1.BaseCard 'Store', T7.CardName 'Store Name' 
from ORIN T0 
Inner Join RIN1 T1 On T1.DocEntry = T0.DocEntry
Inner Join OITM T2 On T2.ItemCode = T1.ItemCode 
Inner Join [@AXE] T3 On T3.Code = T2.U_AXECode
Inner Join RIN12 T4 On T4.DocEntry = T0.DocEntry
Inner Join OCRD T5 On T5.CardCode = T0.CardCode
Inner Join OCRG T6 On T6.GroupCode = T5.GroupCode 
Left Outer Join OCRD T7 On T7.CardCode = T1.BaseCard
Left Outer Join OCRY T8 On T8.Code = T4.CountryB
Inner Join OWHS T9 On T9.WhsCode = T1.WhsCode
Where T0.CANCELED ='N' and T0.DocType ='I' and T1.Price<>0
and T0.TaxDate between @StartDate and @EndDate
```