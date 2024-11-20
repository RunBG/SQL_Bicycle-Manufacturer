# SQL_Bicycle-Manufacturer

## 1. Introduction and Motivation

## 2. The goal of creating this project

## 3. Read and explain dataset

## 4. Data Processing & Exploratory Data Analysis

## 5. Ask questions and solve it
**5.1 Calculate Quantity of items, Sales value & Order quantity by each Subcategory in L12M**

~~~~sql
SELECT 
        FORMAT_DATETIME("%b %Y", sod.ModifiedDate) AS period
      , ps.Name name
      , SUM(sod.OrderQty) AS qty_item
      , SUM(sod.LineTotal) AS total_sales
      , COUNT(DISTINCT sod.SalesOrderID ) AS order_cnt
FROM adventureworks2019.Sales.SalesOrderDetail sod
LEFT JOIN adventureworks2019.Production.Product p
ON p.ProductID=sod.ProductID
LEFT JOIN adventureworks2019.Production.ProductSubcategory ps 
ON  CAST(p.ProductSubcategoryID AS int)=ps.ProductSubcategoryID
WHERE DATE(sod.ModifiedDate) >= (SELECT date_sub(MAX(DATE(ModifiedDate)) , INTERVAL 12 MONTH) FROM adventureworks2019.Sales.SalesOrderDetail)
GROUP BY Name, period
ORDER BY period desc, name asc
~~~~
**RESULT**
|period  |name             |qty_item|total_sales|order_cnt|
|--------|-----------------|--------|-----------|---------|
|Sep 2013|Bike Racks       |312     |22828.512  |71       |
|Sep 2013|Bike Stands      |26      |4134       |26       |
|Sep 2013|Bottles and Cages|803     |4676.562808|380      |
|Sep 2013|Bottom Brackets  |60      |3118.14    |19       |
|Sep 2013|Brakes           |100     |6390       |29       |
|Sep 2013|Caps             |440     |2879.482616|203      |
|Sep 2013|Chains           |62      |752.928    |24       |
|Sep 2013|Cleaners         |296     |1611.8307  |108      |
|Sep 2013|Cranksets        |75      |13955.85   |20       |
|...     |...              |...     |...        |...      |

**5.2 Calc % YoY growth rate by SubCategory & release top 3 cat with highest grow rate. Can use metric: quantity_item. Round results to 2 decimal**
~~~~sql
WITH qty_item_subcategory
AS(
      SELECT 
        FORMAT_DATETIME("%Y", sod.ModifiedDate) AS period
      , ps.Name AS name
      , sum(sod.OrderQty) AS qty_item
FROM adventureworks2019.Sales.SalesOrderDetail sod
LEFT JOIN adventureworks2019.Production.Product p
ON p.ProductID=sod.ProductID
LEFT JOIN adventureworks2019.Production.ProductSubcategory ps 
ON  cast(p.ProductSubcategoryID AS int)=ps.ProductSubcategoryID
GROUP BY 2,1 
ORDER BY 2,1),
cal_prv_qty AS(
SELECT
      name
      ,qty_item
      ,LAG(qty_item) OVER(PARTITION BY name ORDER BY period) prv_qty
FROM qty_item_subcategory)
SELECT      
      name
      ,qty_item
      ,prv_qty
      ,ROUND((qty_item-prv_qty)/prv_qty,2) AS qty_diff
FROM cal_prv_qty
ORDER BY qty_diff DESC
LIMIT 3

~~~~

**RESULT**

|Name    |qty_item         |prv_qty|qty_diff   |
|--------|-----------------|-------|-----------|
|Mountain Frames|3168      |510    |5.21       |
|Socks   |2724             |523    |4.21       |
|Road Frames|5564          |1137   |3.89       |

**5.3 Ranking Top 3 TeritoryID with biggest Order quantity of every year. If there's TerritoryID with same quantity in a year, do not skip the rank number**

~~~~sql
WITH
cal_order_cnt AS(
  SELECT 
      FORMAT_DATETIME("%Y", sod.ModifiedDate) AS period_yr
      ,soh.TerritoryID
      ,sum(sod.OrderQty) AS order_cnt
FROM adventureworks2019.Sales.SalesOrderDetail sod
LEFT JOIN adventureworks2019.Sales.SalesOrderHeader soh
USING(SalesOrderID)
GROUP BY period_yr, TerritoryID ),
rk_by_order_cnt AS(
  SELECT 
      period_yr
      ,TerritoryID
      ,order_cnt
      ,DENSE_RANK() OVER(PARTITION BY period_yr ORDER BY order_cnt DESC) AS rk
FROM cal_order_cnt
ORDER BY period_yr DESC)
SELECT 
       period_yr
      ,TerritoryID
      ,order_cnt
      ,rk
FROM rk_by_order_cnt
WHERE rk <=3
~~~~

**RESULT**

|period_yr|TerritoryID      |order_cnt|rk         |
|---------|-----------------|---------|-----------|
|2014     |4                |11632    |1          |
|2014     |6                |9711     |2          |
|2014     |1                |8823     |3          |
|2013     |4                |26682    |1          |
|2013     |6                |22553    |2          |
|2013     |1                |17452    |3          |
|2012     |4                |17553    |1          |
|2012     |6                |14412    |2          |
|2012     |1                |8537     |3          |
|2011     |4                |3238     |1          |
|2011     |6                |2705     |2          |
|2011     |1                |1964     |3          |

**5.4 Calc Total Discount Cost belongs to Seasonal Discount for each SubCategory**

~~~~sql
SELECT
  EXTRACT(year FROM sod.ModifiedDate) AS year
  ,ps.name AS subcategory
  ,ROUND(SUM(sod.OrderQty*sod.UnitPrice*so.DiscountPct),2)  AS total_cost
FROM `adventureworks2019.Sales.SalesOrderDetail` sod
LEFT JOIN `adventureworks2019.Sales.SpecialOffer`  so
ON sod.SpecialOfferID = so.SpecialOfferID
LEFT JOIN `adventureworks2019.Production.Product`  p
ON sod.ProductID = p.ProductID
LEFT JOIN `adventureworks2019.Production.ProductSubcategory`  ps
ON CAST(p.ProductSubcategoryID AS INT) = ps.ProductSubcategoryID
WHERE LOWER(so.type) LIKE '%seasonal discount%'
GROUP BY 1,2
~~~~

**RESULT**

|year|subcategory                  |total_cost|
|----|-----------------------------|----------|
|2012|Helmets                      |827.65    |
|2013|Helmets                      |1606.04   |

**5.5 Retention rate of Customer in 2014 with status of Successfully Shipped (Cohort Analysis)**

~~~~sql
WITH 
raw_data_1 AS(
  SELECT
    EXTRACT(month FROM ShipDate) AS month_join
    ,CustomerID
    ,COUNT(SalesOrderID) AS sales_cnt
  FROM `adventureworks2019.Sales.SalesOrderHeader`
  WHERE EXTRACT(year FROM ShipDate) = 2014 AND status = 5
  GROUP BY 1,2
  ORDER BY 2),
raw_data_2 AS(
  SELECT
    *
    ,ROW_NUMBER() OVER(PARTITION BY customerid ORDER BY month_join) AS num
  FROM raw_data_1
  ORDER BY 1),

  first_order AS(
  SELECT
    raw_data_2.month_join AS month_order,
    CustomerID
  FROM raw_data_2
  WHERE num = 1
  ORDER BY 2),
  all_join AS(
  SELECT
    r1.month_join,
    r1.CustomerID,
    f.month_order,
    CONCAT('M - ',r1.month_join - f.month_order) AS month_diff,
    r1.sales_cnt
  FROM raw_data_1 AS r1
  LEFT JOIN first_order AS f ON r1.customerID = f.customerID
  ORDER BY 2)

SELECT
  month_join
  ,month_diff
  ,COUNT(customerID) AS customer_cnt
FROM all_join
GROUP BY 1,2
ORDER BY 1
~~~~
**RESULT**
|month_join|month_diff|customer_cnt|
|----------|----------|-----------|
|1         |M - 0     |2076       |
|2         |M - 1     |78         |
|2         |M - 0     |1805       |
|3         |M - 0     |1918       |
|3         |M - 2     |89         |
|3         |M - 1     |51         |
|4         |M - 3     |252        |
|4         |M - 2     |61         |
|4         |M - 1     |43         |
|...       |...       |...        |

**5.6 Trend of Stock level & MoM diff % by all product in 2011. If %gr rate is null then 0. Round to 1 decimal**
~~~~sql
WITH raw_data AS (
  SELECT
     p.Name AS name
    ,EXTRACT(year FROM w.ModifiedDate) AS yr
    ,EXTRACT(month FROM w.ModifiedDate) AS mth
    ,SUM(w.StockedQty) AS Stock_curr
  FROM `adventureworks2019.Production.Product`AS p
  LEFT JOIN `adventureworks2019.Production.WorkOrder` AS w ON p.ProductID = w.ProductID
  WHERE EXTRACT(year FROM w.ModifiedDate) = 2011
  GROUP BY 1,2,3
  ORDER BY 1,2,3 DESC
  ),
  pre_result AS(
  SELECT
     name
    ,yr
    ,mth
    ,Stock_curr
    ,LEAD(Stock_curr) OVER(PARTITION BY Name ORDER BY mth DESC) AS Stock_prv
  FROM raw_data
  ORDER BY 1)
SELECT
   *
  ,CASE
            WHEN (100.0 * (Stock_curr - Stock_prv)/Stock_prv) IS NULL THEN 0
          ELSE
          ROUND((100.0 * (Stock_curr - Stock_prv)/Stock_prv),1)
   END
  AS diff
FROM pre_result
~~~~

**RESULT**
|name|yr                           |mth   |Stock_curr |Stock_prv|diff|
|----|-----------------------------|------|-----------|---------|----|
|BB Ball Bearing|2011                         |12    |8475       |14544    |-41.7|
|BB Ball Bearing|2011                         |11    |14544      |19175    |-24.2|
|BB Ball Bearing|2011                         |10    |19175      |8845     |116.8|
|BB Ball Bearing|2011                         |9     |8845       |9666     |-8.5 |
|BB Ball Bearing|2011                         |8     |9666       |12837    |-24.7|
|BB Ball Bearing|2011                         |7     |12837      |5259     |144.1|
|BB Ball Bearing|2011                         |6     |5259       |         |0    |
|Blade          |2011                         |12    |1842       |3598     |-48.8|
|Blade          |2011                         |11    |3598       |4670     |-23  |
|...          |...                         |...    |...       |...     |...  |

**5.7 Calc Ratio of Stock / Sales in 2011 by product name, by month**

~~~~sql
WITH 
  sale_info AS(
    SELECT 
          FORMAT_DATETIME("%m", sod.ModifiedDate) AS month
        , FORMAT_DATETIME("%Y", sod.ModifiedDate) AS year
        , p.ProductID 
        , p.Name
        , SUM(sod.OrderQty) AS sales_cnt
  FROM adventureworks2019.Sales.SalesOrderDetail sod
  LEFT JOIN adventureworks2019.Production.Product p
  ON p.ProductID=sod.ProductID
  GROUP BY 1,2,3,4),
  stock_info AS(
    SELECT 
          FORMAT_DATETIME("%m", ModifiedDate) AS month
        , FORMAT_DATETIME("%Y", ModifiedDate) AS year
        , ProductID 
        , SUM(StockedQty) AS stocks_cnt
    FROM adventureworks2019.Production.WorkOrder
    GROUP BY 1,2,3
  )
  SELECT 
        sale.month
        ,sale.year
        ,sale.ProductID
        ,sale.Name
        ,sale.sales_cnt
        ,stock.stocks_cnt
        ,ROUND(stock.stocks_cnt/sale.sales_cnt,1) AS ratio
  FROM sale_info sale
  LEFT JOIN stock_info stock ON sale.ProductID=stock.ProductID AND sale.month=stock.month AND sale.year=stock.year
  ORDER BY 1 DESC, 2, 7 DESC
~~~~

**RESULT**
|month|year                         |ProductID|Name       |sales_cnt|stocks_cnt|ratio|
|-----|-----------------------------|---------|-----------|---------|----------|-----|
|12   |2011                         |745      |HL Mountain Frame - Black, 48|1        |27        |27   |
|12   |2011                         |743      |HL Mountain Frame - Black, 42|1        |26        |26   |
|12   |2011                         |748      |HL Mountain Frame - Silver, 38|2        |32        |16   |
|12   |2011                         |722      |LL Road Frame - Black, 58|4        |47        |11.8 |
|12   |2011                         |747      |HL Mountain Frame - Black, 38|3        |31        |10.3 |
|12   |2011                         |726      |LL Road Frame - Red, 48|5        |36        |7.2  |
|12   |2011                         |738      |LL Road Frame - Black, 52|10       |64        |6.4  |
|12   |2011                         |730      |LL Road Frame - Red, 62|7        |38        |5.4  |
|12   |2011                         |741      |HL Mountain Frame - Silver, 48|5        |27        |5.4  |
|12   |2011                         |725      |LL Road Frame - Red, 44|12       |53        |4.4  |
|12   |2011                         |729      |LL Road Frame - Red, 60|10       |43        |4.3  |
|12   |2011                         |732      |ML Road Frame - Red, 48|10       |16        |1.6  |
|12   |2011                         |750      |Road-150 Red, 44|25       |38        |1.5  |
|...   |...                         |...      |...|...       |...        |...  |


**5.8 No of order and value at Pending status in 2014**
~~~~sql
SELECT 
    EXTRACT (YEAR FROM ModifiedDate) AS yr
    , Status
    , COUNT(distinct PurchaseOrderID) AS order_Cnt 
    , ROUND(SUM(TotalDue),3) AS value
FROM `adventureworks2019.Purchasing.PurchaseOrderHeader`
WHERE Status = 1
AND EXTRACT(YEAR FROM  ModifiedDate) = 2014
GROUP BY 1,2
~~~~
**RESULT**
|yr |Status                       |order_Cnt|value      |
|---|-----------------------------|---------|-----------|
|2014|1                           |224      |3873579.012|


## 6. Conclusion
