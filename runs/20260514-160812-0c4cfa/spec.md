use this skill to create a medallion architecture: https://github.com/microsoft/skills-for-fabric/tree/main/skills/e2e-medallion-architecture

split the notebooks per layer, so 1 for bronze, 1 for silver, 1 for gold

In silver I want to add fields for ingestion_dt and source_dt
In gold I want to have the following tables:
Dim_Customer -> join customer, customer address, address and leave all relevant/meaningfull fields
Dim_Product -> join product, productcategory, producdescription, productmodel, productmodelproductdescription. Please notice that in productcategory there is a self join. So in the end I have the category (which is the parent category) and a subcategory.
Dim_Sales --> Use the salesperson field from salesorderheader, but remove the adventureworks/ in front of the name and remove the last number at the end off the name
Fact_Sales --> join salesorderdetail and salesorderheader and keep all relevant fields

Use only one onelake for bronze silver and gold

Create a semantic model on top off the golden layer where you can count the distinct salespersons. Also I want to know which salespersons sell the most, but also give the most discounts. Create some reports on top off this!
