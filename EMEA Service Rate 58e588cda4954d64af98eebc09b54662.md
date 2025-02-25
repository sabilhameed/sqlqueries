# EMEA Service Rate

Function: EMEA
Database: SAR, SGME, SME
Server: azepw-sql001

```html
SELECT	--B.DocEntry ,C.BaseEntry , B.BaseLine ,C.BaseLine,
          A.DocNum 'PO No.', 
		--A.DocDate 'PO Date',
		  A.[DocDate], (convert(char(3) , 
		  A.[DocDate], 0)) 'Month', convert(char(2), 
		  A.[DocDate], 11) 'Year', 
		  A.CardCode 'Vendor Code', 
		  A.CardName 'Vendor Name',
	   -- B1.NumAtCard 'Vendor Ref No.',
		  B.ItemCode, 
		  B.Dscription 'Item Description', 
		  B2.ItmsGrpNam 'Item Group Name',
		  B1.U_ABCC 'ABC Classification', 
		  B1.CodeBars 'Bar Code',
		  B1.U_AXE 'AXE Name', 
		  B1.U_BrandName 'Brand Name', 
		  B1.U_PrdCatLineName 'PrdCatLineNum', 
		  B.Quantity 'PO Qty', 
		  B.Price 'PO Unit Price',
		  A.DocTotal 'PO Total Value',
		  B.LineTotal 'PO Line Value', 
		  ISNULL(C.QUANTITY,0) 'APRes Qty', 
		  ISNULL(C.LINETOTAL,0) 'APRes Line Value',
		  ISNULL(C.QUANTITY,0)*100/B.Quantity 'SR Qty%', 
		  ISNULL(C.LINETOTAL,0)*100/(case when isnull(B.LineTotal,0)=0 then 1 else isnull(B.LineTotal,0) end) 'SQ Value%'

FROM	OPOR A 
		
		INNER JOIN POR1 B ON A.DocEntry = B.DocEntry --and a.DocEntry=2659
		INNER JOIN OITM B1 ON B.ItemCode=B1.ItemCode
		INNER JOIN OITB B2 ON B1.ItmsGrpCod=B2.ItmsGrpCod
		--INNER JOIN OPCH B3 ON A.
		LEFT OUTER JOIN (SELECT	BaseEntry,BaseLine,SUM(QUANTITY) QUANTITY,SUM(LINETOTAL) LINETOTAL
FROM	PCH1 
		WHERE	BaseType='22' --and baseentry=2659
		GROUP BY BaseEntry,BaseLine) C ON B.DocEntry = C.BaseEntry AND B.LineNum = C.BaseLine  
		WHERE  b1.ItmsGrpCod = 100 and  CONVERT(VARCHAR(8),A.DocDate,112) BETWEEN '20240101' AND '20241231'
```