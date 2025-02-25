# ATP (Available to Promise ) Query

Function: Inventory
Server: azepw-sql001

This document contains a SQL query for an ATP (Available to Promise) function related to inventory, specifically for a server named azepw-sql001. The query includes information such as item number, item name, item group, inventory levels, pending purchase orders, in-transit items, and more. The SQL code is included in a code block for reference.

```sql
Select T0.ItemCode 'Item No.', T0.FrgnName 'GlobalCode', T0.ItemName,T0.U_KWT_MOHDesc_EN,T0.U_KWT_MOHDesc_AR, T1.ItmsGrpNam 'Item Group', T0.U_MISCode 'Collections', 
T0.u_AXECode 'AXE',T0.U_AXE as'AXE NAME', T0.U_Dim4PrdLine 'Dim4 Product Line',T0.[U_SublineBusName], T0.U_BINLocation 'WHs BIN Locations', 
STK.InStock, T0.IsCommited 'Commited', T0.OnOrder, 
Isnull(T0.OnOrder-TX.OpenAPR,0) 'Pending PO', Isnull(TX.OpenAPR,0) 'In-Transit', 
(T0.OnHand - T0.IsCommited) 'ATP QTY', (T0.OnHand+T0.OnOrder-T0.IsCommited) 'Net on Hand', STk.AvgPrice 'Total Cost', 
(Select AvgPrice = CASE When Stk.InStock > 0 then (Stk.AvgPrice / stk.InStock) else 0 end) 'AvgPrice',
Isnull(PP9001.Currency,'') 'Price Currency', PP9001.Price,   

 

Isnull(T0.U_SMEMatStatus,'') 'SME Mat.Status',  

 

---- SME Material Status Name
(Select A1."Descr" from CUFD A0 Left Outer Join UFD1 A1 On A1."TableID" = A0."TableID" and A1."FieldID" = A0."FieldID"
Where A0."TableID" = 'OITM' and A0."AliasID" = 'SMEMatStatus' and A1."FldValue" = T0."U_SMEMatStatus") "SME Mat.Status Desc.",

 

U_SMEMATSTATDATE,

 

Isnull(T0.U_SEMatStatus,'') 'SE Mat.Status',

 

---- SE Material Status Name
(Select A1."Descr" from CUFD A0 Left Outer Join UFD1 A1 On A1."TableID" = A0."TableID" and A1."FieldID" = A0."FieldID"
Where A0."TableID" = 'OITM' and A0."AliasID" = 'SEMatStatus' and A1."FldValue" = T0."U_SEMatStatus") "SE Mat.Status Desc.",

 

 

isnull(T0.U_CountryOrg,'') 'Country of Origin', 
isnull(T0.U_BarCode,'') 'Bar Code', isnull(T0.[U_KSA_eCosma],'') 'KSA_eCosma',T0.SuppCatNum 'Custom Code',
Isnull(T0.U_PrdHierarchy,'') 'Product Hierarchy', TXM.GRPODate 'GRPO Date', t0.CreateDate  'Creation Date', 
---- Brand, Product Line, Product Subline
(Select [Name] from [@BRAND] Where [Code] = T0.U_BrandCode) 'Brand',
(Select distinct([U_LnName]) from [@PRODCAT] Where [U_LnCode] = T0.U_PrdCatLnCode and [U_SubLnCode]= T0.U_PrdCatSubLnCode and [U_BrandCode]=T0.U_BrandCode) 'Product Line',
(Select distinct([U_SubLnName]) from [@PRODCAT] Where [U_LnCode] = T0.U_PrdCatLnCode and [U_SubLnCode]= T0.U_PrdCatSubLnCode and [U_BrandCode]=T0.U_BrandCode) 'Product Sub Line',
----
(Select L1Name =    Case 
                        When T0.ItmsGrpCod = 100 Then (Select distinct([U_L1Name]) from [@PRODCLASSALE] Where [U_L1Code] = T0.U_PrdClsSalL1Code)
                        When T0.ItmsGrpCod = 101 Then (Select distinct([U_L1Name]) from [@PRODCLASPOSM] Where [U_L1Code] = T0.U_PrdClsPosL1Code) 
                    End) 'L1 Name',
---- 
(Select L2Name =    Case 
                        When T0.ItmsGrpCod = 100 Then (Select distinct([U_L2Name]) from [@PRODCLASSALE] Where [U_L1Code] = T0.U_PrdClsSalL1Code and [U_L2Code] = T0.U_PrdClsSalL2Code)
                        When T0.ItmsGrpCod = 101 Then (Select distinct([U_L2Name]) from [@PRODCLASPOSM] Where [U_L1Code] = T0.U_PrdClsPosL1Code and [U_L2Code] = T0.U_PrdClsPosL2Code) 
                    End) 'L2 Name',
---- 
(Select L3Name =    Case 
                        When T0.ItmsGrpCod = 100 Then (Select distinct([U_L3Name]) from [@PRODCLASSALE] Where [U_L1Code] = T0.U_PrdClsSalL1Code and [U_L2Code] = T0.U_PrdClsSalL2Code and [U_L3Code] = T0.U_PrdClsSalL3Code)
                        When T0.ItmsGrpCod = 101 Then (Select distinct([U_L3Name]) from [@PRODCLASPOSM] Where [U_L1Code] = T0.U_PrdClsPosL1Code and [U_L2Code] = T0.U_PrdClsPosL2Code and [U_L3Code] = T0.U_PrdClsPosL3Code) 
                    End) 'L3 Name'
---- 
from OITM T0 Inner Join OITB T1 on T0.ItmsGrpCod = T1.ItmsGrpCod 
  Left Outer Join 
  (
  --Select 100 OpenQty, A1.ItemCode
  --from OPCH A0 Inner Join PCH1 A1 On A1.DocEntry = A0.DocEntry
  --Where A0.isIns ='Y' and A1.LineStatus ='O'
  --Group By A1.ItemCode
      Select A0.ItemCode, A0.FrgnName, 
    (Select Isnull(SUM(A1.OpenQty),0) from PCH1 A1 Inner Join OPCH A2 On A2.DocEntry = A1.DocEntry 
    Where A1.ItemCode = A0.ItemCode and A2.isIns ='Y' and A1.LineStatus ='O' ) 'OpenAPR'
    from OITM A0
  ) TX On TX.ItemCode=T0.ItemCode
left outer join

 

(
      Select M0.ItemCode, M0.FrgnName, 
    (Select Isnull(max(M2.[DocDate]),0) from PDN1 M1 Inner Join OPDN M2 On M2.DocEntry = M1.DocEntry 
    Where M1.ItemCode = M0.ItemCode  GROUP BY M1.ItemCode) 'GRPODate'
    from OITM M0
  ) TXM On TXM.ItemCode=T0.ItemCode

 

 

  --- OnHand Stock from SWH001, SWH004, SWH007
  Left Outer Join 
  (
  Select SUM(A1.OnHand) InStock, A1.ItemCode, SUM(A1.OnHand * A1.AvgPrice) AvgPrice
  from OITW A1 
  Where A1.WhsCode in ('SWH001','SWH002','SWH007')
  Group By A1.ItemCode
  ) STK On T0.ItemCode=STK.ItemCode
  --- Item Price from Price List No.30
  Left Outer Join
  (Select ItemCode, Price, Currency from ITM1 Where PriceList =30
  ) PP9001 On T0.ItemCode = PP9001.ItemCode

  Where T0.ItemType ='I'
```

```tsx
Select 

T0.ItemCode 'Item No.', 
T0.FrgnName 'GlobalCode', 
T0.ItemName, 
T0.U_KWT_MOHDesc_EN,
T0.U_KWT_MOHDesc_AR,
T1.ItmsGrpNam 'Item Group', 
T0.U_MISCode 'Collections', 
T0.u_AXECode 'AXE',
T0.U_AXE as'AXE NAME', 
T0.U_Dim4PrdLine 'Dim4 Product Line',
T0.[U_SublineBusCode], 
T0.[U_SublineBusName], 
T0.U_BINLocation 'WHs BIN Locations', 
STK.InStock, 
STK.IsCommited 'Commited', 
T0.OnOrder, 
Isnull(T0.OnOrder-TX.OpenAPR,0) 'Pending PO', 
Isnull(TX.OpenAPR,0) 'In-Transit', 
(STK.InStock - STK.IsCommited) 'ATP QTY', 
(T0.OnHand+T0.OnOrder-T0.IsCommited) 'Net on Hand', 
STk.AvgPrice 'Total Cost', 
(Select AvgPrice = CASE When Stk.InStock > 0 then (Stk.AvgPrice / stk.InStock) else 0 end) 'AvgPrice',
Isnull(PP9001.Currency,'') 'Price Currency', 
PP9001.Price,   
Isnull(T0.U_SMEMatStatus,'') 'SME Mat.Status',  
---- SME Material Status Name
(Select A1."Descr" from CUFD A0 Left Outer Join UFD1 A1 On A1."TableID" = A0."TableID" and A1."FieldID" = A0."FieldID"
Where A0."TableID" = 'OITM' and A0."AliasID" = 'SMEMatStatus' and A1."FldValue" = T0."U_SMEMatStatus") "SME Mat.Status Desc.", 
U_SMEMATSTATDATE,
Isnull(T0.U_SEMatStatus,'') 'SE Mat.Status', 
---- SE Material Status Name
(Select A1."Descr" from CUFD A0 Left Outer Join UFD1 A1 On A1."TableID" = A0."TableID" and A1."FieldID" = A0."FieldID"
Where A0."TableID" = 'OITM' and A0."AliasID" = 'SEMatStatus' and A1."FldValue" = T0."U_SEMatStatus") "SE Mat.Status Desc.", 
isnull(T0.U_CountryOrg,'') 'Country of Origin', 
isnull(T0.U_BarCode,'') 'Bar Code', 
isnull(T0.[U_KSA_eCosma],'') 'KSA_eCosma',
T0.SuppCatNum 'Custom Code',
Isnull(T0.U_PrdHierarchy,'') 'Product Hierarchy', 
TXM.GRPODate 'GRPO Date', 
t0.CreateDate  'Creation Date', 
---- Brand, Product Line, Product Subline
(Select [Name] from [@BRAND] Where [Code] = T0.U_BrandCode) 'Brand',
(Select distinct([U_LnName]) from [@PRODCAT] Where [U_LnCode] = T0.U_PrdCatLnCode and [U_SubLnCode]= T0.U_PrdCatSubLnCode and [U_BrandCode]=T0.U_BrandCode) 'Product Line',
(Select distinct([U_SubLnName]) from [@PRODCAT] Where [U_LnCode] = T0.U_PrdCatLnCode and [U_SubLnCode]= T0.U_PrdCatSubLnCode and [U_BrandCode]=T0.U_BrandCode) 'Product Sub Line',
----
(Select L1Name =	Case 
						When T0.ItmsGrpCod = 100 Then (Select distinct([U_L1Name]) from [@PRODCLASSALE] Where [U_L1Code] = T0.U_PrdClsSalL1Code)
						When T0.ItmsGrpCod = 101 Then (Select distinct([U_L1Name]) from [@PRODCLASPOSM] Where [U_L1Code] = T0.U_PrdClsPosL1Code) 
					End) 'L1 Name',
---- 
(Select L2Name =	Case 
						When T0.ItmsGrpCod = 100 Then (Select distinct([U_L2Name]) from [@PRODCLASSALE] Where [U_L1Code] = T0.U_PrdClsSalL1Code and [U_L2Code] = T0.U_PrdClsSalL2Code)
						When T0.ItmsGrpCod = 101 Then (Select distinct([U_L2Name]) from [@PRODCLASPOSM] Where [U_L1Code] = T0.U_PrdClsPosL1Code and [U_L2Code] = T0.U_PrdClsPosL2Code) 
					End) 'L2 Name',
---- 
(Select L3Name =	Case 
						When T0.ItmsGrpCod = 100 Then (Select distinct([U_L3Name]) from [@PRODCLASSALE] Where [U_L1Code] = T0.U_PrdClsSalL1Code and [U_L2Code] = T0.U_PrdClsSalL2Code and [U_L3Code] = T0.U_PrdClsSalL3Code)
						When T0.ItmsGrpCod = 101 Then (Select distinct([U_L3Name]) from [@PRODCLASPOSM] Where [U_L1Code] = T0.U_PrdClsPosL1Code and [U_L2Code] = T0.U_PrdClsPosL2Code and [U_L3Code] = T0.U_PrdClsPosL3Code) 
					End) 'L3 Name'
---- 
from OITM T0 Inner Join OITB T1 on T0.ItmsGrpCod = T1.ItmsGrpCod 
  Left Outer Join 
  (
  --Select 100 OpenQty, A1.ItemCode
  --from OPCH A0 Inner Join PCH1 A1 On A1.DocEntry = A0.DocEntry
  --Where A0.isIns ='Y' and A1.LineStatus ='O'
  --Group By A1.ItemCode
  	Select A0.ItemCode, A0.FrgnName, 
	(Select Isnull(SUM(A1.OpenQty),0) from PCH1 A1 Inner Join OPCH A2 On A2.DocEntry = A1.DocEntry 
	Where A1.ItemCode = A0.ItemCode and A2.isIns ='Y' and A1.LineStatus ='O' ) 'OpenAPR'
	from OITM A0
  ) TX On TX.ItemCode=T0.ItemCode
left outer join

 (
  	Select M0.ItemCode, M0.FrgnName, 
	(Select Isnull(max(M2.[DocDate]),0) from PDN1 M1 Inner Join OPDN M2 On M2.DocEntry = M1.DocEntry 
	Where M1.ItemCode = M0.ItemCode  GROUP BY M1.ItemCode) 'GRPODate'
	from OITM M0
  ) TXM On TXM.ItemCode=T0.ItemCode

  --- OnHand Stock from SWH001, SWH004, SWH007
  Left Outer Join 
  (
  Select SUM(A1.OnHand) InStock,SUM(A1.IsCommited) as IsCommited, A1.ItemCode, SUM(A1.OnHand * A1.AvgPrice) AvgPrice
  from OITW A1 
  Where A1.WhsCode in ('SWH001','SWH002')
  Group By A1.ItemCode
  ) STK On T0.ItemCode=STK.ItemCode
  --- Item Price from Price List No.30
  Left Outer Join
  (Select ItemCode, Price, Currency from ITM1 Where PriceList =30
  ) PP9001 On T0.ItemCode = PP9001.ItemCode
  
  Where T0.ItemType ='I'
```