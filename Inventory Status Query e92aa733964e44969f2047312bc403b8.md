# Inventory Status Query

Function: Inventory
Database: SGME, SME

```sql
DECLARE @AsOnDate DATE = GETDATE();

-- Temporary table creation
CREATE TABLE #Inventory_Status
(
    ItemCode NVARCHAR(50),
    ItemName NVARCHAR(200),
    Warehouse NVARCHAR(10),
    ItmsGrpNam NVARCHAR(50),
    AXECode NVARCHAR(16),
    AXEName NVARCHAR(30),
    InStockQty NUMERIC(19,6),
    InStockValue NUMERIC(19,6),
    InTransitQty NUMERIC(19,6),
    InTransitValue NUMERIC(19,6),
    PriceCurrency NVARCHAR(5),
    Price NUMERIC(19,6),
    CountryOfOrigin NVARCHAR(10),
    Barcode NVARCHAR(16),
    GRPODate DATE,
    ApDocDate DATE,
    Brand NVARCHAR(50),
    ABCClassification NVARCHAR(50),
    ProductLine NVARCHAR(50)
);

-- Data insertion
INSERT INTO #Inventory_Status
    SELECT 
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
        ISNULL(T1.U_CountryOrg,''),
        ISNULL(T1.U_BarCode,''), 
        TXM.GRPODate,
        APTX.APResDate,
        (SELECT [Name] FROM [@BRAND] WHERE [Code] = T1.U_BrandCode),
        T1.U_ABCC,
        (SELECT DISTINCT([U_LnName]) FROM [@PRODCAT] WHERE [U_LnCode] = T1.U_PrdCatLnCode AND [U_SubLnCode] = T1.U_PrdCatSubLnCode AND [U_BrandCode] = T1.U_BrandCode)
    FROM OITW T0 
    INNER JOIN OITM T1 ON T1.ItemCode = T0.ItemCode 
    INNER JOIN OITB T2 ON T2.ItmsGrpCod = T1.ItmsGrpCod
    LEFT OUTER JOIN
    (
        SELECT 
            M0.ItemCode, 
            M0.Warehouse, 
            SUM(M0.InQty) - SUM(M0.OutQty) AS BalQty, 
            SUM(M0.TransValue) AS Value
        FROM OINM M0 
        WHERE M0.DocDate <= @AsOnDate
        GROUP BY M0.ItemCode, M0.Warehouse 
    ) STK ON STK.ItemCode = T0.ItemCode AND STK.Warehouse = T0.WhsCode
    LEFT OUTER JOIN
    (
        SELECT 
            A1.ItemCode, 
            A1.WhsCode, 
            ISNULL(SUM(A1.OpenQty),0) AS APResQty, 
            ISNULL(SUM((A1.LineTotal / A1.Quantity) * A1.OpenQty),0) AS APResVal
        FROM PCH1 A1 
        INNER JOIN OPCH A2 ON A2.DocEntry = A1.DocEntry
        WHERE
		A2.isIns ='Y' AND A1.LineStatus ='O' AND A2.DocDate <= @AsOnDate AND A2.CANCELED = 'N'
GROUP BY A1.ItemCode, A1.WhsCode
) APRes ON APRes.ItemCode = T0.ItemCode AND APRes.WhsCode = T0.WhsCode
LEFT OUTER JOIN
(
SELECT
A1.ItemCode,
A1.WhsCode,
ISNULL(SUM(A1.Quantity),0) AS GRNQty,
ISNULL(SUM(A1.LineTotal),0) AS GRNVal
FROM OPDN A0
INNER JOIN PDN1 A1 ON A1.DocEntry = A0.DocEntry
INNER JOIN OPCH A2 ON A2.DocEntry = A1.BaseEntry
WHERE A0.DocDate > @AsOnDate AND A2.isIns = 'Y' AND A2.DocDate <= @AsOnDate AND A0.CANCELED = 'N'
GROUP BY A1.ItemCode, A1.WhsCode
) GRN ON GRN.ItemCode = T0.ItemCode AND GRN.WhsCode = T0.WhsCode
LEFT OUTER JOIN
(
SELECT ItemCode, Price, Currency FROM ITM1 WHERE PriceList = 30
) PP9001 ON T0.ItemCode = PP9001.ItemCode
LEFT OUTER JOIN
(
SELECT M0.ItemCode,
(
SELECT ISNULL(MAX(M2.DocDate),0)
FROM PDN1 M1
INNER JOIN OPDN M2 ON M2.DocEntry = M1.DocEntry
WHERE M1.ItemCode = M0.ItemCode AND M2.DocDate <= @AsOnDate
GROUP BY M1.ItemCode
) AS GRPODate
FROM OITM M0
) TXM ON TXM.ItemCode = T0.ItemCode
LEFT OUTER JOIN
(
SELECT P0.ItemCode,
(
SELECT ISNULL(MAX(X2.DocDate),0)
FROM PCH1 X1
INNER JOIN OPCH X2 ON X2.DocEntry = X1.DocEntry
WHERE X1.ItemCode = P0.ItemCode AND X2.DocDate <= @AsOnDate
GROUP BY X1.ItemCode
) AS APResDate
FROM OITM P0
) APTX ON APTX.ItemCode = T0.ItemCode
WHERE ItmsGrpNam = 'SELLABLE'
AND (ISNULL(STK.BalQty,0) + ISNULL(APRes.APResQty,0) + ISNULL(GRN.GRNQty,0)) > 0
ORDER BY ItmsGrpNam, ItemCode, Warehouse;

-- Retrieval of data from temporary table
SELECT * FROM #Inventory_Status;

-- Temporary table cleanup
DROP TABLE #Inventory_Status
```