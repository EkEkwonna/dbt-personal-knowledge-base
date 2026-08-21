First thing we did after opening Snowflake is:
On the left we saw: 
- Catalog >> Database Explorer
- pressed "+" for new SQL File

And we have the following: 

~~~~sql
create warehouse transforming;
create database raw;
create database analytics;
create schema raw.jaffle_shop;
create schema raw.stripe;
~~~~


- Database - overall container where data will live
- Database - where folders will live
- Schemas - the actual folders themselves

- - - - - - - - - - - 

We then create a table:

~~~~sql
create table raw.jaffle_shop.customers (
    id integer, 
    first_name varchar, 
    last_name varchar

)
~~~~


and we can then INSERT data from a public S3 bucket : 

~~~~sql
copy into raw.jaffle_shop.customers (id, first_name, last_name)
from 's3://dbt-tutorial-public/jaffle_shop_customers.csv'
file_format= (
    type = 'CSV',
    field_delimiter = ',',
    skip_header = 1
)
~~~~

We have a separate table for orders : 

~~~~sql
create table raw.jaffle_shop.orders (
    id integer, 
    user_id integer,
    order_date date, 
    status varchar, 
    etl_loaded_at timestamp default current_timestamp
)
~~~~

Copy in table from S3 bucket 

~~~~sql 
copy into raw.jaffle_shop.orders (id, user_id, order_date,status)
from 's3://dbt-tutorial-public/jaffle_shop_orders.csv'
file_format = (
    type = 'CSV',
    field_delimiter = ',',
    skip_header = 1
    )
~~~~

And the final one for payments 

~~~~sql 
create table raw.stripe.payments (
    id integer, 
    orderid integer,
    paymentmethod varchar,
    status varchar,
    amount integer, 
    created date, 
    _batched_at timestamp default current_timestamp
)

copy into raw.stripe.payments (id,orderid, paymentmethod, status, amount,created)
from 's3://dbt-tutorial-public/stripe_payments.csv'
file_format = (
    type = 'CSV',
    field_delimiter = ',',
    skip_header = 1
    )
~~~~

### Snowflake Account details 

To locate your Snowflake Account details you will find an ALPHANUMERIC code when you click on your Snowflake profile.<br>Alternatively you will find it at the top of the url, contained between forward slashes. 

Alternatively go to : 
* Profile > Account > View Account Detailst (copy your account identifier)

The second part refers to your account  i.e. JB1234 in the account identifier : XXXXXX-JB1234



### dbt 

When creating the new environment you have to establish a connection. <br> To ensure we connect to Snowflake you can select Type Snowflake and enter:
* Account identifier 
* Database 
* Warehouse 
