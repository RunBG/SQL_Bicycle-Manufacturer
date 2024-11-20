# SQL_Bicycle-Manufacturer

## 1. Introduction and Motivation

## 2. The goal of creating this project

## 3. Read and explain dataset

## 4. Data Processing & Exploratory Data Analysis

## 5. Ask questions and solve it
**5.1 Calc Quantity of items, Sales value & Order quantity by each Subcategory in L12M**

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
|...|...        |...      |...   |...       |

## 6. Conclusion
