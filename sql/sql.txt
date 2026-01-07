use database bison_db;
use warehouse bison_wh;
create or replace schema amazon_vendor_schema;

create or replace table stg_invoice_items as
select * from AMAZON_VENDOR_ORDER_TO_CASH__SAMPLE.PUBLIC."InvoiceItems";

create or replace table stg_calendar as
select * from AMAZON_VENDOR_ORDER_TO_CASH__SAMPLE.PUBLIC."AmazonCalendar";

create or replace table stg_catalog as
select * from amazon_vendor_order_to_cash__sample.public."Catalog";

create or replace table stg_payments as
select * from amazon_vendor_order_to_cash__sample.public."Payments";

create or replace table stg_shortage_claims as
select * from amazon_vendor_order_to_cash__sample.public."ShortageClaims";

create or replace table stg_price_claims as
select * from amazon_vendor_order_to_cash__sample.public."PriceClaims";

select * from stg_price_claims;


create or replace table dim_product as(
select
row_number() over (order by ASIN, "Product") as id_product,
ASIN,
"Product" as Product,
"Brand" as Brand,
"CategoryPath" as CategoryPath,
"Model" as Model,
"ReleaseDate" as ReleaseDate,
"MAP",
UPC,
EAN,
GTIN,
"AmazonLastPrice" as AmazonLastPrice,
MSRP,
URL,
"ItemUID" as ItemUID,
"Status" as Status
from (select ASIN,"Product","Brand","CategoryPath","Model",UPC,EAN,GTIN,"AmazonLastPrice",MSRP,"Status","ReleaseDate","ItemUID","MAP",URL from stg_catalog)
);


create or replace table dim_location as(
select distinct
row_number() OVER (order by "CountryCode", "Marketplace") as location_id,
"CountryCode" as CountryCode,
"Marketplace" as Marketplace,
"Company" as Company,
"Payee" as Payee
from (select distinct "CountryCode","Marketplace","Company","Payee" from stg_invoice_items)
);

create or replace table dim_payments as(
select
row_number() over (order by "PaymentNumber", "AcctType") as payment_id,
"PaymentNumber" as PaymentNumber,
"PaymentStatus" as PaymentStatus,
"PaymentType" as PaymentType,
"PaymentVoidedReason" as PaymentVoidedReason,
"Acct" as Acct,
"AcctType" as AcctType,
"AcctPath" as AcctPath,
"Description" as Description,
"PaymentCurrency" as PaymentCurrency,
"InvoiceCurrency" as InvoiceCurrency,
"InvoiceNumber" as InvoiceNumber
from (select distinct "PaymentNumber","PaymentStatus","PaymentType","PaymentVoidedReason","Description","PaymentCurrency", "Acct", "AcctType", "AcctPath", "InvoiceCurrency", "InvoiceNumber" from stg_payments)
);


create or replace table dim_date as(
select distinct
"Date" as Date,
"Year" as Year,
"Month" as Month,
"Day" as Day,
"Qtr" as Qtr,
"Week" as Week,
"DayofYear" as DayofYear,
"DayofWeek" as DayofWeek,
"DayofQtr" as DayofQtr,
"FiscalYear" as FiscalYear,
"FiscalMonth" as FiscalMonth,
"FiscalDay" as FiscalDay,
"FiscalQtr" as FiscalQtr, 
"FiscalWeek" as FiscalWeek,
"FiscalDayofYear" as FiscalDayofYear,
"FiscalDayofWeek" as FiscalDayofWeek,
"FiscalDayofQtr" as FiscalDayofQtr
from stg_calendar
);

create or replace table fact_invoice_items as(
select
row_number() over (order by i."InvoiceDate", i."InvoiceNumber") as id_invoiceitems,
dp.id_product,
dd.DATE as Date,  
pm.id_payment, 
dloc.location_id as id_location,
i."InvoiceNumber",
i."PurchaseOrder",
i."CreationDate",
i."PaymentDate",
i."ReceivedASINs" as ASIN,
i."InvoiceQty",
i."ReceivedQty",
i."ShortageQty",
i."UnitCost",
(i."InvoiceQty" * i."UnitCost") as Line_Amount,
i."Currency",

sc.ShortageAmount,
pc.CostVariance,
pc.DefectAmount,
pc.MatchedQty,
pc.POQty,
pc.UnitPriceClaim,

pay.AmountPaid,
pay.RemainingAmount,
pay.TermsDiscountTaken,
pay.WithholdingAmount,

row_number() over (partition by i."InvoiceNumber" order by i."ReceivedASINs") as InvoiceItemSequence,
lag(i."UnitCost") over (partition by i."ReceivedASINs" order by i."InvoiceDate") as PreviousUnitCost


from stg_invoice_items i
left join (select distinct DATE from dim_date) dd on i."InvoiceDate" = dd.DATE 
    
left join (select "Marketplace", "CountryCode", MAX(location_id) as location_id
           from dim_location
           group by 1, 2) 
           dloc on i."Marketplace" = dloc."Marketplace" and i."CountryCode" = dloc."CountryCode"

left join (select "InvoiceNumber" as JOIN_KEY, 
           SUM("AmountPaid") as AmountPaid, 
           MAX("RemainingAmount") as RemainingAmount, 
           SUM("TermsDiscountTaken") as TermsDiscountTaken, 
           SUM("WithholdingAmount") as WithholdingAmount
           from stg_payments 
           group by 1)
           pay on i."InvoiceNumber" = pay.JOIN_KEY

left join(select "InvoiceNumber" as JOIN_KEY, "ASIN" as ASIN_KEY, SUM("ShortageAmount") as ShortageAmount
          from stg_shortage_claims 
          group by 1, 2) 
          sc on i."InvoiceNumber" = sc.JOIN_KEY and i."ReceivedASINs" = sc.ASIN_KEY

left join(select "InvoiceNumber" as JOIN_KEY, "ASIN" as ASIN_KEY, 
          SUM("CostVariance") as CostVariance, 
          SUM("DefectAmount") as DefectAmount, 
          SUM("MatchedQty") as MatchedQty, 
          MAX("POQty") as POQty, 
          MAX("POUnitPrice") as UnitPriceClaim
          from stg_price_claims 
          group by 1, 2) 
          pc on i."InvoiceNumber" = pc.JOIN_KEY and i."ReceivedASINs" = pc.ASIN_KEY

left join (select InvoiceNumber as JOIN_KEY, MAX(payment_id) as id_payment
           from dim_payments
           group by 1) 
           pm ON i."InvoiceNumber" = pm.JOIN_KEY

left join (select "ASIN" as ASIN_KEY, MAX(id_product) as id_product
           from dim_product
           group by 1) 
           dp ON i."ReceivedASINs" = dp.ASIN_KEY);
