# Freight Report

Function: Freight
Database: SGME, SME
Server: azepw-sql001

```sql
SELECT T0.[DocNum] as 'Doc No', 
       T0.[DocDate], 
       (convert(char(3), T0.[DocDate], 0)) 'Month', 
       convert(char(2), T0.[DocDate], 11) 'Year', 
       T2.[U_BrandName] as'Brand', 
       T1.[ItemCode], 
       T1.[Dscription], 
       T1.[Quantity], 
       T0.[Ref1]as 'Shipment No',
       T3.[ItmsGrpNam] as 'Item Group', 
       T1.[FobValue]as 'Total AC', 
       T1.[TtlExpndLC] as ' Freight Cost', 
       T1.[Quantity]*T4.[Price]as 'Purchase SC', 
       (T1.[TtlExpndLC]/(T1.[Quantity]*T4.[Price]))*100 as 'Freght %'
FROM [dbo].[OIPF] T0 
INNER JOIN [dbo].[IPF1] T1 ON T0.[DocEntry] = T1.[DocEntry] 
INNER JOIN [dbo].[OITM] T2 ON T1.[ItemCode] = T2.[ItemCode] 
INNER JOIN [dbo].[OITB] T3 ON T2.[ItmsGrpCod] = T3.[ItmsGrpCod] 
LEFT OUTER JOIN [dbo].[ITM1] T4 ON T1.[ItemCode] = T4.[ItemCode] 
LEFT OUTER JOIN [dbo].[OPDN] T5 ON T5.DocEntry=T1.BaseEntry AND T1.BaseType='20' 
LEFT OUTER JOIN [dbo].[OSHP] T6 ON T5.[TrnspCode]=T6.TrnspCode 
WHERE T0.[DocDate] >= '2023-06-01' AND T0.[DocDate] <= '2024-06-01'
  AND T4.[PriceList] = 70 
  AND T4.[Price] > 0 
  AND T2.[InvntItem] ='Y' 
ORDER BY T0.[DocNum]

```