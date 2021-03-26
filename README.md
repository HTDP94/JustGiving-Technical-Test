# JustGiving-Technical-Test

My solution to the JustGiving Data Engineer Technical Test

## How to build and run solution

1. Save the zip file 'JustGiving_Technical_Test.zip' in a folder located on the same server that your instance of SQL Server is running (or somewhere the SSIS package can access).
2. Unzip this file. The contents should be three folders called 'SSIS', 'SQL' and 'Download'.
3. Open the SSIS package in Visual Studio (located at '\SSIS\JustGiving Interview Project - Harry Palmer\JustGiving Interview Project - Harry Palmer.sln' in the zipped file) and set the string variable 'ProjectFolder' to the path of the folder the zipped file was saved to in step 1.
4. In the connection manager, edit the OLE DB connection 'AdventureWorks' to point at the AdventureWorks database on your instance of SQL Server.
5. Run the SQL script '010_Initialise_Environment.sql' loacted in the SQL folder to build the relevant tables the package will use.
6. Run the SSIS package to build the desired fact table in the AdventureWorksDW database named FactUSOrders.
7. Run the SQL script 'output_query.sql' to produce the desired output query.

## Describe and explain reasoning behind any assumptions made

* I've assumed that the dimension tables in AdventureWorksDW are up to date and corect (such as DimProduct and DimDate). I believe refreshing these tables and ensuring they contain all the required data would be beyond the scope of this project.
* I've assumed that all price fields on the 'Sales.SalesOrderDetail' table are in USD.
* My solution to the problem involves downloading the results of the API into a string, and then performing string manipulation on this to turn it into a CSV, so my solution assumes that the format of this API won't change.
* I noticed a 'Status' field on the 'Sales.SalesOrder' table but I couldn't find a relevant lookup table to explain it's values. As a result I;ve assumed all orders are to be included in the final fact table.

## Explain why you decided to implement things the way that you did

Given the exchange rate between USD and NZD changes over time, I felt a more complete approach to this solution would be to consider what the exchange rate was on the date the order placed, as this would give you a better idea of the value of the order in NZD at the time it was placed. As a result, instead of getting the latest exchange rate data from exchangratesapi.io, I instead got the historical exchange rate data between '2011-05-31' and '2014-06-30' (as this was the date range of orders in the Sales.SalesOrderHeader table) and used this to calculate an orders value in NZD.

This historic exchange rate data is processed in the 'Get NZD Exchange Rate' container in the SSIS package: 
* First a script task (written in c#) downloads this data into a string, then inserts a row into a csv containing the 'Date' and 'ExchangeRate' fields.
* Next this csv is loaded into a loading table (Loading.NZDExchangeRates) in the 'AdventureWorks' database.
* Finally this data is cleaned and processed into the existing Sales.CurrencyRate table in the SQL script '020_Clean_Exchange_Rate_Data.sql'. When implementing this I realised that the history API wasn't returning all the requested data, with 338 days between '2011-05-31' and '2014-06-30' not being returned with an exchange rate. In orde not to miss orders placed on these days, I wrote some logic to set the exchange rate for these missing days to exchange rate of the previous day (or the most recent day for which data was provided).

On refletion, this may have been a slight overcomplication of the task (which was further complicated by the issue with missing dates) however it felt like a reasonable thing to consider. In a work environment I would have discussed this with other team members before going ahead, as maybe using the latest data would have sufficed.

## How the solution could be made better

1. The FactUSOrders table that I've have built contains price fields in both USD and NZD, however if we wanted to consider a different currency it would require adding more fields to the fact table, and so doesn't scale well. A better approach would be to have another fact table containing historic exchange rates which could then be updated and refreshed in a seperate process. We could then remove the NZD price fields from the FactUSOrders table and instead include a CurrencyCode which we can then use to join to this FactCurrencyRate table. This approach would make adding other currencies much easier.
2. Given the results of the API are provided in a JSON string, I think the solution would be better if it dealt with this data using a JSON parser. I tried to do this briefly but couldn't seem to get it to work, however this approach would make this package a bit more robust in dealing with changes to the format of the API. Currently any change would almost certainly break it.




