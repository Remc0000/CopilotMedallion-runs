In silver I want to add fields for ingestion_dt and source_dt
In gold I want to have the following tables:
Dim_Customer -> join customer, customer address, address and leave all relevant/meaningfull fields
Dim_Product -> join product, productcategory, producdescription, productmodel, productmodelproductdescription
Dim_Sales --> Use the salesperson field from salesorderheader, but remove the adventureworks/ in front of the name and remove the last number at the end off the name
Fact_Sales --> join salesorderdetail and salesorderheader and keep all relevant fields

Create a semantic model on top off the golden layer where you can count the distinct salespersons. Also I want to know which salespersons sell the most, but also give the most discounts. Create some reports on top off this!