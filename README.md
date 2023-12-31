
# Sales Data Analysis

Sales data analysis using [MYSQL](https://dev.mysql.com/downloads/mysql/) & [Looker studio](https://lookerstudio.google.com/overview).With the help of MYSQL we are going to clean the data and using Looker studio we are going to build interactive dashboard.

Reference : [codebasics yt](https://www.youtube.com/watch?v=hhZ62IlTxYs&list=PLeo1K3hjS3uva8pk1FI3iK9kCOKQdz1I9&index=1)

Data source : download [db_dumb.sql](https://codebasics.io/resources/sales-insights-data-analysis-project) from this link or refer this [sales sheets](https://docs.google.com/spreadsheets/d/1YNBWXCBhPZjQtNwEiMuxERYd-78WN4TmAUKegxo4Cek/edit?usp=sharing)




# Dashboard

- [Looker Studio Dashboard](https://lookerstudio.google.com/reporting/e7a6149d-1384-4484-84d5-f5bcd4dc1922)

![1) Dashboard](https://github.com/VedantMalgundkar/Simple-linear-regression-with-custom-gradient-descent-/assets/129035372/836461ff-0546-4426-a308-25ac9d4eb8dc)


# Setup

Import "db_dumb.sql" into MYSQL Workbench.
```
C:\WINDOWS\system32>mysql -u USERNAME -pPASSWORD

mysql> CREATE DATABASE sales;
mysql> USE sales;
mysql> source PATH OF your file/db_dump.sql;
mysql> SHOW TABLES;
+-----------------+
| Tables_in_sales |
+-----------------+
| customers       |
| date            |
| markets         |
| products        |
| transactions    |
+-----------------+
5 rows in set (0.00 sec)

```



# Data model

![data model](https://github.com/VedantMalgundkar/Simple-linear-regression-with-custom-gradient-descent-/assets/129035372/5c314a28-0d65-4876-b6b1-fa2a2126011f)

This data model follows star schema in which sales transaction is fact table and other tables connected to fact table are dimension tables.


# Preprocessing in sheets

We have transaction data from 2017-2020 & in transaction table we have table as currency in which we have INR & USD.

- we are preparing USD price data for each possible transaction date in transactions table.

So first we will convert all USD transaction to INR by this historical [USD price data](https://finance.yahoo.com/quote/INR%3DX/history?period1=1506816000&period2=1596153600&interval=1d&filter=history&frequency=1d&includeAdjustedClose=true) between 2 oct 2017 to 30 jun 2020.

We are going to fill null values and values for missing dates with last obesevation carried forward(LOCF) method.  

- open downloaded csv file in sheets & keep 'Date' and 'Adj close' column and remove every thing else.
- create third column "Date" and paste following to create range of dates from 4 oct 2017 to 26 jun 2020.
```
=ARRAYFORMULA(TO_DATE(TEXT(ROW(INDIRECT("A"&DATEVALUE("2017-10-04")&
                                       ":A"&DATEVALUE("2020-06-26"))), "mm/d/yyyy")))

```
- now create fourth column 'USD_price_na'with following formula :
```
=VLOOKUP(DATEVALUE(C2),$A$2:$B$718,2,False)

```
- Now in the last 'USD_price' column paste following formula to fill all null & NA values with LOCF method :

```
=IF(OR(ISNA(D2),ISTEXT(D2)),E1,D2)

```
- download as csv
![sheets preprocessing](https://github.com/VedantMalgundkar/Simple-linear-regression-with-custom-gradient-descent-/assets/129035372/d668d05f-e52d-48d4-a279-850f2eb986de)

- Refer this [sheet](https://docs.google.com/spreadsheets/d/1qeL7MAu8ibUYrjg-G0UOLlOfBsCkrH78OIe8UI0eirU/edit?usp=sharing)





# Data transformation using SQL

Import downloaded csv into MYSQL

- create table 'usdtoinr' in sales db

```
CREATE TABLE IF NOT EXISTS usdtoinr(
Date TEXT,
USD_price DOUBLE
);

```
- Import data into "usdtoinr" using following command :

```
LOAD DATA INFILE 'C:/ProgramData/MySQL/MySQL Server 8.0/Uploads/usdtoinr - INR=X.csv'
INTO TABLE usdtoinr
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\r\n'
IGNORE 1 LINES
(@col1,@col2,@col3,@col4,@col5) 
set Date = @col3,USD_price= @col5;

```


Join transactions table and usdtoinr table on the basis of 'order_date'of transactions table and date in 'usdtoinr'

```
CREATE TABLE IF NOT EXISTS new_transactions AS 
SELECT 
t.product_code, t.customer_code,
t.market_code, t.order_date,
t.sales_qty, t.sales_amount,
t.currency, p.USD_price
FROM transactions AS t
JOIN (SELECT STR_TO_DATE(Date,'%m/%d/%Y') AS Date,USD_price AS USD_price FROM usdtoinr) as p
ON t.order_date = p.Date
ORDER BY t.order_date;

```
Now we have USD price for each transaction date.

![transactions transition](https://github.com/VedantMalgundkar/Simple-linear-regression-with-custom-gradient-descent-/assets/129035372/eaa46768-69d3-48b8-aebd-b7e882acb534)


```
SELECT currency,COUNT(currency) FROM new_transactions GROUP BY currency;

```
We can see the following output of groupby query :

![distinct currency](https://github.com/VedantMalgundkar/Simple-linear-regression-with-custom-gradient-descent-/assets/129035372/9fcfae85-be28-412f-b8fb-47224438cd83)

There are four diffrent currency types.

Records With INR currency are duplicate records as the occured less frequently than 'INR\r'.

Same goes for USD.
So we will consider only those currency with maximum frequency.ie('INR\r' & 'USD\r')

 
## Market table transformation

- Before moving further download this file [Loc_coords.csv](https://drive.google.com/file/d/1EHNwi86NjVjVKwnxzJCL3ZWnFwwTRbyF/view?usp=drive_link). we will use this coordinates for visualization.

- we will create new table to store locations

```
CREATE TABLE IF NOT EXISTS loc_coords(
markets_code TEXT,
location TEXT
);

```
- Now we will load downloaded csv to above newly created table.

```
LOAD DATA INFILE 'C:/ProgramData/MySQL/MySQL Server 8.0/Uploads/Loc_coords_.csv'
INTO TABLE loc_coords
FIELDS TERMINATED BY ';'
LINES TERMINATED BY '\r\n'
IGNORE 1 LINES;

```

- Now lets create 'new_markets' table with new column 'location'

```
CREATE TABLE IF NOT EXISTS new_markets(
SELECT 
m.markets_code, m.markets_name,
m.zone, REPLACE(l.location,'"','') AS location
FROM markets as m
JOIN loc_coords AS l
ON m.markets_code = l.markets_code);

```

![market transition](https://github.com/VedantMalgundkar/Simple-linear-regression-with-custom-gradient-descent-/assets/129035372/56748a5d-078d-4c8b-a8e4-1917aca6bac1)




# Data cleaning
- In this step we will remove records which has sales amount less than equal to zero.

- And we will also remove duplicate records in from 'new_transactions'.

![duplicates records](https://github.com/VedantMalgundkar/Simple-linear-regression-with-custom-gradient-descent-/assets/129035372/e0cf88c8-89ff-4afa-b38e-cd30a3af4473)

- Now in this step we will convert ALL USD transactions to INR.

```
CREATE TABLE IF NOT EXISTS cleaned_transactions( 
SELECT *,
(case
WHEN currency = 'USD\r' THEN ROUND(sales_amount*USD_price,0)
ELSE sales_amount
END) AS all_inr_sales_amount
FROM new_transactions WHERE sales_amount !=0 AND sales_amount !=-1 AND currency = 'INR\r' OR currency ='USD\r'
)

```
In newly created cleaned_transactions table we don't have any duplicate records

![no dupli + usd to inr ](https://github.com/VedantMalgundkar/Simple-linear-regression-with-custom-gradient-descent-/assets/129035372/c0a3fb87-97a6-4ce9-abe6-803aa2186e60)

Database overview : 

![Without_whole](https://github.com/VedantMalgundkar/Simple-linear-regression-with-custom-gradient-descent-/assets/129035372/ab7dfbec-123f-4a2a-a80e-e1f323f83d20)





# Data Analysis

1) Top markets by revenue


```
SELECT m.markets_name,SUM(t.all_inr_sales_amount) AS rev 
FROM cleaned_transactions AS t 
LEFT JOIN markets AS m ON t.market_code=m.markets_code 
GROUP BY t.market_code
ORDER BY rev DESC;

```
![rev by markets](https://github.com/VedantMalgundkar/Simple-linear-regression-with-custom-gradient-descent-/assets/129035372/4e100fb5-3de2-4341-b7f4-378df9a888f6)

2) Top customers by revenue
```
SELECT c.custmer_name,SUM(t.all_inr_sales_amount) AS rev 
FROM customers AS c 
JOIN cleaned_transactions AS t 
ON c.customer_code = t.customer_code 
GROUP BY c.custmer_name 
ORDER BY rev DESC; 
```

![top customers](https://github.com/VedantMalgundkar/Simple-linear-regression-with-custom-gradient-descent-/assets/129035372/80a2d4e7-9e0d-426b-a49d-5ae931783f8e)


3) Total sales of year 2020
```
SELECT SUM(all_inr_sales_amount) 
FROM cleaned_transactions 
WHERE order_date LIKE '2020%'; 	-- => 41,36,87,163
```

4) Total sales of month feb-2020.
```
SELECT SUM(all_inr_sales_amount) FROM cleaned_transactions WHERE order_date LIKE '2020-02-__'; -- => 2,56,56,567 
```

5) Total sales of from market 'Delhi NCR'
```
SELECT SUM(all_inr_sales_amount)
FROM cleaned_transactions AS td 
JOIN markets AS mkt ON td.market_code = mkt.markets_code 
WHERE markets_name = 'Delhi NCR'; -- => 51,95,62,163 
```

6) Top product by revenue 

```
SELECT product_code,SUM(all_inr_sales_amount) AS total_rev 
FROM cleaned_transactions 
GROUP BY product_code 
ORDER BY total_rev DESC;
```

![top products](https://github.com/VedantMalgundkar/Simple-linear-regression-with-custom-gradient-descent-/assets/129035372/2b08c80d-b573-4a34-9851-fe8b04716947)


# Data aggregation

This step involves aggreating data from each table as it is nessaccery for dashboard visualization in 'Looker studio'

```
CREATE TABLE IF NOT EXISTS whole_joined_data AS 
SELECT 
t.product_code,
t.customer_code,
t.market_code,
t.order_date,
t.sales_qty,
t.all_inr_sales_amount,
c.custmer_name,
c.customer_type,
d.cy_date,
REPLACE(REPLACE(d.date_yy_mmm, '\n', ''), '\r', '') AS date_yy_mmm,
d.month_name,
d.year,
m.markets_name,
m.location,
m.zone
FROM cleaned_transactions AS t
JOIN customers AS c
ON t.customer_code = c.customer_code
JOIN date AS d
ON t.order_date = d.date
JOIN new_markets AS m
ON t.market_code = m.markets_code;

```

![Whole tables 123](https://github.com/VedantMalgundkar/Simple-linear-regression-with-custom-gradient-descent-/assets/129035372/ee408001-3dc4-444b-a711-19358a76da83)

- Export aggregated table to csv

![Export whole data](https://github.com/VedantMalgundkar/Simple-linear-regression-with-custom-gradient-descent-/assets/129035372/12ae88dc-bc2b-42df-865f-1a2d694134ff)

## Looker Studio Dashboard

- Upload above aggregated csv to google sheets.

- Then we can use that sheet as a data source for visualization in Looker Studio 

- [Looker Studio Dashboard](https://lookerstudio.google.com/reporting/e7a6149d-1384-4484-84d5-f5bcd4dc1922)