# dbt Analytics Engineering Knowledge Base

A comprehensive reference guide covering core dbt concepts, DAG design principles, staging conventions, automated testing, and documentation best practices.

## Table of Contents
- [1.Snowflake Setup](#Snowflake-Setup)
- [2.Connecting dbt to a Snowflake database](#Connecting-dbt-to-a-Snowflake-database)
- [3.Admins vs Developers](#Admins-vs-Developers)
- [4.Connecting dbt to a Snowflake database](#Connecting-dbt-to-a-Snowflake-database)
- [5.Admins vs Developers](#Admins-vs-Developers)
- [6.Deploying a Production Environment](#Deploying-a-Production-Environment)
- [7.Building Models](#Building-Models)
- [8.Referencing Sources](#Referencing-Sources)
- [9.Testing](#Testing)
- [10.DBT Build Command](#DBT-Build-Command)
- [11.Documentation](#Documentation)
- [12.Deployment](#Deployment)



## Snowflake Setup

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

#Snowflake Account details 

To locate your Snowflake Account details you will find an **ALPHANUMERIC** code when you click on your Snowflake profile.

Alternatively you will find it at the top of the url, contained between forward slashes. 

Alternatively go to : 
* Profile > Account > View Account Detailst (copy your account identifier)

The second part refers to your account  i.e. JB1234 in the account identifier : `XXXXXX-JB1234`



### dbt 

When creating the new environment you have to establish a connection. <br> To ensure we connect to Snowflake you can select Type Snowflake and enter:
* Account identifier 
* Database 
* Warehouse 

## Connecting dbt to a Snowflake database

As Snowflake has DEPRECATED the use of single-factor password sign ins we are dependent on using safer methods of congigurating accounts to connections. 

### Key Pair Authentication 

Step 1 - Encrypt a private key 

(Option A: with passphrase - more secure)

~~~~bash
openssl genrsa 2048 | openssl pkcs8 -topk8 -v2 des3 -out rsa_private_key.p8
~~~~

(Option B: withouth passpharese )

~~~~bash
openssl genrsa 2048 | openssl pkcs8 -topk8 -nocrypt -out rsa_private_key.p8
~~~~

Then we generate a publick key from the private key 

~~~~bash 
openssl rsa -in rsa_private_key.p8 -pubout -out rsa_public_key.pub
~~~~


The Snowflake user account is configured with the private key 

~~~~sql
ALTER USER <username> SET RSA_PUBLIC_KEY='<public_key_string>';
~~~~


The private key is stored securely on dbt to allow the handshake. 

---

If using option A. On the terminal (with ```OpenSSL``` installed) it will request a passkey ensure to store this. 

Once produced we should have an a PRIVATE KEY key ```rsa_private_key.p8```  and a public key ```rsa_public_key.pub``` we can use ```cat``` command to review it. 


---

### dbt Target Schema 

The target schema is the schema that the tables created in dbt will fall in. Try to make this a unique schema so it doesn't ovewrite (if in Production)

Managed Repository 

Is managed by dbt and all the code is stored. Will store the code. 

## Admins vs Developers
###   DBT for Developers

Here is a custom design set up for dbt 

1. Set up Databases

~~~~sql
use role sysadmin;
create database raw;
create database analytics;
~~~~

2. Set up Warehouses 

Note: 3 warehouses for different purposes : 
* Loading 
* Transforming 
* Reporting 

~~~~sql 
create warehouse loading
    warehouse_size = xsmall
    auto_suspend = 3600
    auto_resume = false
    initially_suspended = true;

create warehouse transforming
    warehouse_size = xsmall
    auto_suspend = 60
    auto_resume = true
    initially_suspended = true;

create warehouse reporting
    warehouse_size = xsmall
    auto_suspend = 60
    auto_resume = true
    initially_suspended = true;
~~~~

3. Set up Roles & Warhouse permissions

Note: Role specified to each warehouse according

~~~~sql
use role securityadmin;

create role loader;
grant all on warehouse loading to role loader; 

create role transformer;
grant all on warehouse transforming to role transformer;

create role reporter;
grant all on warehouse reporting to role reporter;
~~~~

---

4. Users and Service accounts

Now EVERY user and Service account needs a separate user and the user needs tob e assigned to a role 


~~~~sql 
create user stitch_user -- or fivetran_user
    password = '_generate_this_'
    default_warehouse = loading
    default_role = loader; 

create user claire -- or amy, jeremy, etc.
    password = '_generate_this_'
    default_warehouse = transforming
    default_role = transformer
    must_change_password = true;

create user dbt_cloud_user
    password = '_generate_this_'
    default_warehouse = transforming
    default_role = transformer;

create user looker_user -- or mode_user etc.
    password = '_generate_this_'
    default_warehouse = reporting
    default_role = reporter;

-- then grant these roles to each user
grant role loader to user stitch_user; -- or fivetran_user
grant role transformer to user dbt_cloud_user;
grant role transformer to user claire; -- or amy, jeremy
grant role reporter to user looker_user; -- or mode_user, periscope_user
~~~~

---

5. Loader Role permissions 

~~~~sql
use role sysadmin;
grant all on database raw to role loader;
~~~~

6. Transformer Role permissions

We grant the transformer the power to read the raw data in the database raw AND future objects created from the database raw 

~~~~sql
grant usage on database raw to role transformer;
grant usage on future schemas in database raw to role transformer;
grant select on future tables in database raw to role transformer;
grant select on future views in database raw to role transformer;
~~~~

We also give access to any CURRENT objects in the raw database 


~~~~sql 
grant usage on all schemas in database raw to role transformer;
grant select on all tables in database raw to role transformer;
grant select on all views in database raw to role transformer;
~~~~

Lastly transformers need to be able to create in the analytics database 


~~~~sql 
grant all on database analytics to role transformer;
~~~~

---

7. Let Reporters read transformed data 

~~~~sql
grant usage on database analytics to role reporter;
grant usage on future schemas in database analytics to role reporter;
grant select on future tables in database analytics to role reporter;
grant select on future views in database analytics to role reporter;
~~~~

And any current objects 

~~~~sql
grant usage on all schemas in database analytics to role reporter;
grant select on all tables in database analytics to role reporter;
grant select on all views in database analytics to role reporter;
~~~~


In the end you should end up with something along the lines of : 

![](/Base/Images/example-snowflake-role.png)



---

If once you click on Snowflake Workspace you only see the ```PUBLIC``` role then you have the minimal level of access

If you are a developer you may need the ```TRANSFORMATION``` role 

---

To access a dbt project (if utilising GitHub & Snowflake) you want to ensure you first : 

* Have Snowflake access with the right role 
* Can see your Org and Teams repository on your GitHub

Now in dbt select the following : 

* (Lower Left) Username >> Your Profile
* Scroll to linked account and connect them 

---

###   DBT for Admins

Prerequisites: 
* Need access to Snowflake ```ACCOUNTADMIN``` role 
* Or BOTH ```SYSADMIN``` and ```SECURITYADMIN```


Raw databases are like the Bronze Layer. 

---

To modify different roles you can either adjust the top left hand corner or use 


~~~~sql
USE ROLE SYSADMIN
~~~~

We then create a Raw Database and a Target Database 


~~~~sql 
CREATE DATABASE raw -- usually a RAW data source is already created
CREATE DATABASE analytics_1
~~~~


Then we can set up a dedicated COMPUTE or warehouse for 

~~~~sql
CREATE DATAWAREHOUSE transform_wh
warehouse_size= small
auto_suspend = 60
auto_resume = true
initially_suspend = true
~~~~

For context : 

* auto_suspend - automatically shuts down a virtual warehouse after it stays idle 
* auto_resume - automatically starts up a suspended warehouse when a new SQL statement/query is applied to it 
* initially_suspend - when ````True```` warehouse is built in 'suspended' state and saves credits (if ```False``` the warehouse starts up and immediately consumes credits while idle )

---

Now use the security admin role to create a role 

~~~~sql
USE ROLE securityadmin;
CREATE ROLE transformer_1;
~~~~

Run similar privilges to the section in Developers for transfors (ensuring ```FULL USAGE``` permission on raw datawarehouse and ```ALL``` permissions on analytics datawarehouse )

Note: When dealing with shared databases you can import privileges 

~~~~sql
GRANT IMPORTED PRIVILEGES ON <SHARED_DATABASE> TO ROLE transformer_1
~~~~


---

Lastly to check you have required access.

Use the Role and Use the Warehouse you intend 

~~~~sql
USE ROLE transformer_1 
USE SECONDARY ROLE NONE -- makes sure you don't have any additional privileges from other roles 

USE WAREHOUSE raw 

select * from raw.jaffle_shop.customer 

USE WAREHOUSE analytics 
CREATE SCHEMA dbt_test_user

CREATE TABLE dbt_test_user.test as (
    select 1 as test_column
)

DROP SCHEMA dbt_test_user
~~~~

---

### Setting up OAuth Integration (SSO) Single-Sign On

To access the Connections: 

* Select your account name > Account Settings > Connections

You can then create a new Snowflake Connection 

When selecting Snowflake SSO as the OATH Authentication method. You get access to new fields


In Snowflake you want to run the following : 

~~~~sql
CREATE OR REPLACE SECURITY INTEGRATION DBT_CLOUD
  TYPE = OAUTH
  ENABLED = TRUE
  OAUTH_CLIENT = CUSTOM
  OAUTH_CLIENT_TYPE = 'CONFIDENTIAL'
  OAUTH_REDIRECT_URI = '<REDIRECT_URI>' 
  OAUTH_ISSUE_REFRESH_TOKENS = TRUE
  OAUTH_REFRESH_TOKEN_VALIDITY = 7776000
  OAUTH_USE_SECONDARY_ROLES = 'IMPLICIT';  -- Required for secondary roles
~~~~

* Redirect URI - You can get this information in dbt (URI is provided when you select SSO oath)


Then you add the following 

~~~~sql 
with

integration_secrets as (
  select parse_json(system$show_oauth_client_secrets('DBT_CLOUD')) as secrets
)

select
  secrets:"OAUTH_CLIENT_ID"::string     as client_id,
  secrets:"OAUTH_CLIENT_SECRET"::string as client_secret
from
  integration_secrets;
~~~~

* This will provide you the client ID and the client secret to enter in the OATH credentials on dbt for SSO


You then receive the optional setting to specify a Role 
* Make sure the user has access to the role itself 

---

The next step is the Environment 

You can set up a Development environment here. 
Or specify whether it's Production (adding Staging, Production flags)

We now need to authenticate from the dbt tool by:
* Clicking on Personal profile>> Settings >>Credentials 

Enter the credentials and the details and make sure the details are provided. 

Double check to make sure the details included are accurate. 

When creating a new Github Repo for dbt a Readme file should not be included as it will cause issues for the project initialisation 

```SYSADMIN``` & ```SECURITYADMIN``` are required to set up databases and roles in Snowflake


## Deploying a Production Environment

Consider the Following Database objcects (for Project Wahala): 
* WAHALA_DEV
* WAHALA_PROD
* WAHALA_RAW
* WAHALA_TEST

First start by creating a Service account on Snowflake 

~~~~sql 
CREATE USER wahala_DBT_PROD_SERVICE_ACCOUNT_USER
    PASSWORD = 'test-password-123'
    COMMENT = 'Service account for dbt in the production (PROD) environment for Project Wahala'
    DEFAULT_WAREHOUSE = WAHALA_PROD_WH
    DEFAULT_ROLE = wahala_DBT_PROD_SERVICE_ACCOUNT_ROLE
    MUST_CHANGE_PASSWORD = FALSE
~~~~

Then you must grant the role to the service account 

~~~~sql 
GRANT ROLE wahala_DBT_TEST_SERVICE_ACCOUNT_ROLE TO USER wahala_DBT_TEST_SERVICE_ACCOUNT_USER

GRANT ROLE wahala_DBT_PROD_SERVICE_ACCOUNT_ROLE TO USER wahala_DBT_PROD_SERVICE_ACCOUNT_USER

~~~~

The role ```wahala_prod_service_account_role``` has access to: 
* read from raw WAHALA_RAW (but not write)
* Create/delete/modify tables/views from WAHALA_PROD

On dbt go to Orchestration >> Environment

You should have a Development type Environment already set up.

Create a New Environment : 
* Name: Production 
* dbt Version (latest) but make sure Environments use the same version 
* Custom branch "production"

Deployment Connection Details should provide : 
* Account (Your account id)
* Role (dbt_DEVELOPER)
* Database (dbt_DEV)
* Warehouse (dbt_DEV_WH)

^ These will be defaults but shoudl not be the same as the Development environment. 

When deploying to production these roles should be completely different and Unique to avoid mixing roles. So maybe : 

Deployment Connection Details should provide : 
* Account (Your account id)
* Role (WAHALA_DBT_PROD_SERVICE_ACCOUNT_ROLE)
* Database (WAHALA_PROD)
* Warehouse (WAHALA_PROD_WH)

^ This is an example of a connection you would associate with the "Wahala" Project

---

### Deployment Credentials

For deployment credentials we use the same username and password as what was created i.e from the above example we would have : 

* Username : wahala_DBT_PROD_SERVICE_ACCOUNT_USER
* Password : test-password-123

And you can save the Environment. 
But to Ensure what we built is correct we can build a quick job.

---

Creating a quick Job to test Prodution Environment:

* Job name: Prodution - Daily 
* Environment : Production (from before)
  - This will configure all the details from the environment that was set up 

For Environment Variables we can provide the variables that can be called in the job 

~~~~shell
dbt build
~~~~

You can then schedule it (We have Crontab schedule)

To validate the credential details work you can Run Now. 

You can then refresh the Snowflake to locate the new job output in the location specified 


---


### Setting up Folders for Data Maturity 

Once you have the File Explorer available modify the models folder to contain the following : 

~~~~
models/
    ├── example
    ├── marts
    │   └── .gitkeep
    └── staging
        └── .gitkeep
~~~~

For Good maturity we have built the following :
* Staging Folder - To organise all source logic (perform simple cleaning on raw data imported via snowflake)
* Marts Folder - Organise all business conformed (Combines staging models to create key business concepts [ _DIM & _FACT table creation] )
* .gitkeep - to check in an empty folder 

---



By Default the **dbt_project.yml** should look like : 

~~~~yml

name: 'my_new_project'
version: '1.0.0'
config-version: 2
profile: 'default'
model-paths: ["models"]
analysis-paths: ["analyses"]
test-paths: ["tests"]
seed-paths: ["seeds"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]
target-path: "target"  # directory which will store compiled SQL files
clean-targets:         # directories to be removed by `dbt clean`
  - "target"
  - "dbt_packages"
models:
  my_new_project:
    # Applies to all files under models/example/
    example:
      +materialized: table

~~~~

We can make the following modifications i.e. the ```name``` and the ```my_new_project``` category (under ```models```)  only applies to the name provided 


You may also consider updating the README.md 

Then we : 
1. Commit changes (add detailed messaged)
2. Set up Pull Request to merge the brach to master. 


Once changes are committed on GitHub. We can update the version Control to "Refresh Git state" to see the changes. 

----

### To set up Folders by Data Domains. 

Consider the following Domains (Teams building their own models): 

* Finance 
* Sales 
* Marketing 

And we want to set up a folder for each team to 

We can create the following folders in the models folder: 

~~~~
models/
    ├── example
    ├── finance
    │   └── .gitkeep
    ├── marketing
    │   └── .gitkeep
    └── sales
        └── .gitkeep
~~~~

We would need to modify the the .yml file 


## Building Models

Added a new model ```customer.sql``` in the models repository : 

~~~~sql 
with customers as (

    select
        id as customer_id,
        first_name,
        last_name

    from raw.jaffle_shop.customers

),

orders as (

    select
        id as order_id,
        user_id as customer_id,
        order_date,
        status

    from raw.jaffle_shop.orders

),

customer_orders as (

    select
        customer_id,

        min(order_date) as first_order_date,
        max(order_date) as most_recent_order_date,
        count(order_id) as number_of_orders

    from orders

    group by 1

),


final as (

    select
        customers.customer_id,
        customers.first_name,
        customers.last_name,
        customer_orders.first_order_date,
        customer_orders.most_recent_order_date,
        coalesce(customer_orders.number_of_orders, 0) as number_of_orders

    from customers

    left join customer_orders using (customer_id)

)

select * from final
~~~~

***MAKE SURE TO SAVE THE FILE!***

We should be able to preview (Table Item Bottom left) to see final view of query 

To run the sql queries in the models directory we type 

~~~
dbt run 
~~~

This will execute all the models in the models directory 

When reviewing the Logs (In the Commands section) 
There's a feature to review the details ```Debug Logs```

Note : a new *view* was created when we ran this query

~~~~sql
create or replace view analytics.dbt_eekwonna.customers
~~~~

This is due to the default materialisation (when the model doesn't specify to ```create or replace view/table```) dbt will create an object based on the yml

---
### Overriding configuration parameters

In the ```my_first_dbt_model.sql``` file a Jinja config block is specified with ```{{config(materialized='table')}}```
: 

**my_first_dbt_model.sql** 

~~~~sql
{{ config(materialized='table') }}

with source_data as (

    select 1 as id
    union all
    select null as id

)

--etc....
~~~~

---

The yml file contains all the global configuration for the project 

**dbt_projets.yml**

~~~~yml
# ...
models:
  my_new_project:
    # Applies to all files under models/example/
    example:
      +materialized: table
~~~~

If you want to override configuration you can use the configuration block in the model (Jinja)

---

If the Jinja config is modified to 

~~~~
{{config(materialized='table')}}
~~~~

***MAKE SURE TO SAVE THE FILE!***

To run just the model by itself: 
* to the left of Build icon 
* Run model 

**dbt command line code** 
~~~~
dbt run --select my_first_dbt_model
~~~~

---

NOTE: If Lineage is currently unavailable 

Type the name of the projects in the search and Update Graph

---

### Modularity

The concept of Modularity is breaking down data models into distinct re-usable building blocks (semantic pieces)

We can BREAK down the ```customers.sql``` into several CTEs. 

In a scenario where want to use the```customers``` (belonging to the ```customers.sql```) 

~~~~sql
-- ...
with customers as (

    select
        id as customer_id,
        first_name,
        last_name

    from raw.jaffle_shop.customers
)
-- ...
~~~~

instead of copying this down into another large SQL model we can break this down and form the dependency

We can take the logic out of the ```customers.sql``` and make it it's own model 

---

### Staging 

Note: dbt Naming conventions suggest that if producing a 1-1 model with source [schema].[object_name] becomes [schema]__[object] 
("_" underscore to replace ".")


By creating the following staging models :


**stg_jaffle_shop__customers.sql**

~~~sql
select
    id as customer_id,
    first_name,
    last_name

from raw.jaffle_shop.customers
~~~

**stg_jaffle_shop__orders.sql**

~~~~sql
select
    id as order_id,
    user_id as customer_id,
    order_date,
    status

from raw.jaffle_shop.orders
~~~~

To reference the staging models in a separate model we use the ```__ref``` function in the dbt sql script

i.e. the customers.sql script becomes 

~~~~sql 

with customers as (

select * from {{ ref('stg_jaffle_shop__customers')}}

),

orders as (

select * from {{ref('stg_jaffle_shop__orders')}}

),

customer_orders as (

    select
        customer_id,

        min(order_date) as first_order_date,
        max(order_date) as most_recent_order_date,
        count(order_id) as number_of_orders

    from orders

    group by 1

),


final as (

    select
        customers.customer_id,
        customers.first_name,
        customers.last_name,
        customer_orders.first_order_date,
        customer_orders.most_recent_order_date,
        coalesce(customer_orders.number_of_orders, 0) as number_of_orders

    from customers

    left join customer_orders using (customer_id)

)

select * from final
~~~~

This will reference the staging models in the final model. 

![](/Base/Images/dbtLineageCustomerModel.png)


---

### Running Jobs based on dependencies

To run every upstream model before a model you can select : 
* Toggle next to Build symbol 
* ```Run +model (Upstream)```

This ensures all the dependencies prior to the model are executed as well. 

note command line syntax : 

~~~~bash
dbt run --select +customers
~~~~

---

### Good Model Naming Practices

#### Sources 

- Tables of raw data loaded into platform 
- Data should be loaded automatically or scheduled 
- Data Engineers are responsible for orchestrating loading process

#### Staging 

- Staging models are built 1:1 with the source
- For each Source Table there should be a source table 
- Make data look the way we wished it came in the data platform 
- Renaming, Restructuring data types , Currency conversions, 

#### Intermediate 

- Intermediate models are were joins and aggregations take place according to business need
- Bridges raw data and final reporting tables (facts & dimensions tables)



Fact tables 

Dimension tables

---

### Cleaning and Reorganizing 

We can Remove the ```models/example``` folder as it's custom default folder 

The folder structure can be udpated to reflect the following : 

~~~~
models
    ├── marts
    │   └── dim_customers.sql
    └── staging
        ├── stg_jaffle_shop__orders.sql
        └── stg_jaffle_shop__customers.sql
~~~~


### Materialization strategies 

To follow best practices when it comes to materialisation we must update the ```dbt_project.yml``` file . 

1. Rename the project
2. Update the models project name 
3. Provide provide default materialisation

**dbt_project.yml**
~~~~yml 
name: 'dbt_jaffle_shop_project'

# ...
models:
    dbt_jaffle_shop_project:
        staging:
            +matarialized: view
        marts:
            +materialized: table

~~~~

This ensures whenever a model is ran inside the staging folder (where the object type is not defined) a view is created for Staging and a table is set up for Marts. 

Note: Marts are typically downstream tables. 

When a mart is queried if it was a view and depending on other views, every view dependency is executed every time the mart view model is ran

---

It's also good practice to break down the sub levels of staging down by different sources. 

i.e. right now we have 1 source system 'Jaffle Shop' so we can add this as a sub folder for staging. 

~~~~
models
    ├── marts
    │   └── marketing
    │       └── dim_customers.sql
    └── staging
        └── jaffle_shop
            ├── stg_jaffle_shop__orders.sql
            └── stg_jaffle_shop__customers.sql
~~~~


Marts we can also break down based on which teams are consuming the necessary data (so marts may also have subfolders like 'Finance', 'Accounting', 'Marketing' etc. )


---
---

## Excercise 

####  Building a fct_orders model 

* Inspect ```raw.stripe.payment```
* Create a ```stg_stripe__payments.sql``` model in `models/staging/stripe`
* Create a `fct_orders.sql` (not stg_orders) model with the following fields. Place this in the `marts/finance` directory:
* order_id
* customer_id
* amount (hint: this has to come from payments)


#### Refactor your dim_customers Model

* Add a new field called lifetime_value to the `dim_customers` model:
* lifetime_value: the total amount a customer has spent at jaffle_shop

Hint: The sum of lifetime_value is $1,672


---
---

## Excercise 

####  Building a fct_orders model 

* Inspect ```raw.stripe.payment```
* Create a ```stg_stripe__payments.sql``` model in `models/staging/stripe`
* Create a `fct_orders.sql` (not stg_orders) model with the following fields. Place this in the `marts/finance` directory:
  * order_id
  * customer_id
  * amount (hint: this has to come from payments)


#### Refactor your dim_customers Model

* Add a new field called lifetime_value to the `dim_customers` model:
* lifetime_value: the total amount a customer has spent at jaffle_shop

Hint: The sum of lifetime_value is $1,672


### Working out 

When investigating the ```raw.stripe.payment``` I came accross the following fields : 

| id| orderid | status | amoung | created |
| :--- | :--- | :--- | :--- | :--- | 
| 1 | 1| Success | 10000 | 2018-01-01 |
| 2 | 2| Success | 20000 | 2018-01-02 |
| 3 | 3| Success | 100 | 2018-01-04 |
| 4 | 4| Fail | 1700 | 2018-01-05 |
| 5 | 4| Success | 1700 | 2018-01-05 |


When staging consider the correct business terms for the staging table that will aid the pipeline going downstream. 

We rename the columns so every column name to be intuitive : 

**stg_stripe__payments.sql**
~~~~sql

with 

source as (

    select * from raw.stripe.payments

),

renamed as (

    select
        id as payment_id,
        orderid as order_id,
        paymentmethod as payment_method,
        status as payment_status,
        amount as payment_amount,
        created as payment_created,
        _batched_at

    from source

)

select * from renamed
~~~~

As the `fct_orders.sql` needs to incorporate the payment we need to consider the status (success/failed) 

To make things easier a defined CTE for each successful order's payment is incorporated in the fact 

**order_payments CTE**
~~~~sql
order_payments as (
    select
        order_id,
        sum (case when payment_status = 'success' then payment_amount end) as amount

    from payments
    group by 1
~~~~
 so that 

**fct_orders**

~~~~sql
with orders as  (
    select * from {{ ref ('stg_jaffle_shop__orders' )}}
),

payments as (
    select * from {{ ref ('stg_stripe__payments') }}
),

order_payments as (
    select
        order_id,
        sum (case when payment_status = 'success' then payment_amount end) as amount

    from payments
    group by 1
),

 final as (

    select
        orders.order_id,
        orders.customer_id,
        orders.order_date,
        coalesce (order_payments.amount, 0) as amount

    from orders
    left join order_payments using (order_id)
)

select * from final
~~~~

Note the renaming of columns allows us to keep consisency with joings and the `final` CTE


orders fact table can now be utilised for the dimension `dim_customer.sql` 

~~~~sql

--customer_id, first_name,last_name
with customers as (

     select * from {{ ref('stg_jaffle_shop__customers') }}

),
-- order_id, customer_id, order_date,status 
orders as ( 

    select * from {{ ref('fct_orders') }}

),customer_orders as (
    select
        customer_id,
        min (order_date) as first_order_date,
        max (order_date) as most_recent_order_date,
        count(order_id) as number_of_orders,
        sum(amount) as lifetime_value
    from orders
    group by 1
),
 final as (
    select
        customers.customer_id,
        customers.first_name,
        customers.last_name,
        customer_orders.first_order_date,
        customer_orders.most_recent_order_date,
        coalesce (customer_orders.number_of_orders, 0) as number_of_orders,
        customer_orders.lifetime_value
    from customers
    left join customer_orders using (customer_id)
)
select * from final
~~~~

With a Lineage that looks as such : 

![](/Base/Images/dimCustomerLineage.png)


#### Useful Resources 

* [dbt Commants](https://docs.getdbt.com/category/list-of-commands?version=2.0)
* [Syntax Overviews](https://docs.getdbt.com/reference/node-selection/syntax?version=2.0)
* [SQL Query Anatomy](https://medium.com/@lomso.dzingwa/the-anatomy-of-a-sql-query-189dd0664851)
* [Subqueries and CTEs](https://medium.com/@datainsights17/subqueries-and-ctes-in-sql-aa1ff4b17686)


#### SQL Query Anatomy 

The query order of execution goes : 

| ORDER | CLAUSE | Function |
| :--- | :--- | :--- |
| 1 | WITH , JOIN , FROM | Constructs base data set| 
| 2 | WHERE | Filters base data | 
| 3 | GROUP BY | Aggregates base data |
| 4 | HAVING | Filters aggregated data |
| 5 | SELECT | Returns Final data |
| 6 | DISTINCT | Removes duplicate rows |
| 7 | ORDER BY | Sorts data |
| 8 | LIMITS | Limits by row count |


## Referencing Sources

In dbt we can utilise the `.yml` file to reference a datasource with the use of language such as 

~~~
{{source('schema_name','object_name')}}
~~~

This will create a direct reference to the data souce.

dbt will use the yml file to identify the reference of the data source. 

This means if we have several models using a specific data source. We can make the change in the yml file and dbt will compile all the models accordingly (instead of making changes manaually)


---
### Updating Staging Models 

When observing the Lineage

![](/Base/Images/dimCustomerLineage.png)

We can see there are 3 staging models (peforming light formatting and renaming on source data to ingest raw data)

If we take `stg_jaffle_shop_orders.sql` as an example we can see the source has been **`HARD CODED`** into the model 

**stg_jaffle_shop_orders**
~~~~sql 
    select 
        id as order_id,
        user_id as customer_id,
        order_date,
        status

    from raw.jaffle_shop.orders
~~~~

`raw.jaffle_shop.orders` is hardcoded, but it's best practice to create a source to later reference in the yml file . 

---

Create a source yml in the same staging subdirectory as the staging models : 

(Note: use of "_src" as prefix to have `yml` file at the top)

~~~~
models
    ├── marts
    │   └── ...
    └── staging
        ├── jaffle_shop
        │   ├── _src_jaffle_shop.yml
        │   ├── stg_jaffle_shop__orders.sql
        │   └── stg_jaffle_shop__customers.sql
        └── stripe
            ├── _src_stripe.yml
            └── stg_stripe_payments.sql
~~~~

for each source object we can either copy from the [developer article](https://docs.getdbt.com/docs/build/sources) (by googling: Add source to your DAG)

~~~
sources:
  - name: jaffle_shop
    database: raw  
    schema: jaffle_shop  
    tables:
      - name: orders
      - name: customers

  - name: stripe
    tables:
      - name: payments
~~~

Alternative you can type `__sources` (Note : 2 undescores) in the yml and a default preset structure will appear

---

**src_jaffle_shop.yml**
~~~~yml
sources:
  - name: jaffle_shop
    database: raw
    schema: jaffle_shop
    tables:
      - name: customers
      - name: orders
~~~~

Once created we can save and refresh Lineage we should expect the following :

![](/Base/Images/srcLineage.png)

**scr_stripe_payments.yml**
~~~~yaml
sources:
  - name: stripe
    database: raw
    schema: stripe
    tables:
      - name: payments
~~~~

Note: that if database and schema details are not udpated dbt will assume the same schema as the name

---

Once the source files have been set up. 
We can connect the source files in the models by using `Jinja` to : 

* `raw.jaffle_shop.orders`  &rarr; `{{source('jaffle_shop','orders')}}`

**stg_jaffle_shop_orders**
~~~~sql
    select 
        id as order_id,
        user_id as customer_id,
        order_date,
        status

    from {{source('jaffle_shop','orders')}}
~~~~

Note: `Jinja` uses the source name and table name (from the `.yml` file) 

The Lineage should update as such: 

![](/Base/Images/stg_ordersLineage.png)

Also it is best practice to have 1 staging model for each source. 

When clicking the `compile` (`</>` Item) we can see the compiled (with the referenced table)

---

### Maintanable 

This means that for any future migrations going forward we would ONLY have to upate the `.yaml` file to update any reference objects

---

### Source Freshness

This allows us to ensure raw data in sources is new. (We can define how fresh we want the data to be at any one time) 

dbt will check this to let us know if data is getting old 


In `_src_jaffle_shop.yml` by adding a `config` and a `__freshness` we can update the `.yml` as such 

*_src_jaffle_shop.yml**
~~~~yml
sources:
  - name: jaffle_shop
    database: raw
    schema: jaffle_shop
    tables:
      - name: customers
      - name: orders
        config:
          freshness:
            warn_after:
              count: 6
              period: hour
            error_after:
              count: 12
              period: hour
            loaded_at_field: etl_loaded_at
~~~~

^ This will test the freshness of the table and provide an alert/error accordingly 

By running 

~~~bash 
dbt source freshness
~~~

Error logs 
**WARNING**
~~~~log
13:50:06 Started source jaffle_shop.orders (external)
13:50:07 Stale [0.0s] source jaffle_shop.orders (external) (last updated 5 days 11h 27m 42s ago)
~~~~

**WARNING**
~~~~log
14:23:47 Warned [0.0s] source jaffle_shop.orders (external) (last updated 5 days 12h 1m 22s ago)
~~~~

We can see the **Stale** or **Warning** message 

---

We can also apply this at Schema level: 

*_src_jaffle_shop.yml**
~~~~yml
sources:
  - name: jaffle_shop
    database: raw
    schema: jaffle_shop
    config:
      freshness:
        warn_after:
          count: 21
          period: day
        error_after:
          count: 22
          period: day
      loaded_at_field: etl_loaded_at
    tables:
      - name: customers
        config:
          freshness: null
      - name: orders

~~~~

NOTE `customers` table doesn't contain an `etl_loaded_at` field hence freshness is udpated to null


---

### Codegen package 

To configure and reference several objects in the data warehouse into our `.yml` 

The codegen package can be installed from the [dbt Hub](http://hub.getdbt.com/) 

By following the instruction in the [Codegen Package](https://hub.getdbt.com/dbt-labs/codegen/latest/) site

We can see that we need to create a `.yml` in the project folder with the following : 

~~~~yml
packages:
  - package: dbt-labs/codegen
    version: 0.14.1
~~~~

(Ensuring we have dbt version between `1.1.0` and `3.0.0`) 

Run the following to install the package

~~~~bash 
dbt deps
~~~~

This will allow us to use the new macro to install. Now we can use a macro to generate all the sources. 

In the usage section we can see a line to print all the sources. 

We can run this in a new file to compile a list of all the objects linked to the source provided

~~~~
{{ codegen.generate_source(schema_name= 'jaffle_shop', database_name= 'raw') }}
~~~~

`Compile` returns 

~~~~yml
version: 2

sources:
  - name: jaffle_shop
    database: raw
    tables:
      - name: customers
      - name: orders
~~~~

This will provide all the sources that we have in our warehouse for the specified `raw` database and `jaffle_shop` schema 

We can save this in a new yml file and store this as our `_src_jaffle_shop.yml` file 

---

#### Generating Staging models

Once you have the `.yml` copied from the COMPILED code from codegen. We can generate the staging models by reveiwing the `.yml`

Above each object we should have the option to generate a staging model

![](/Base/Images/GenerateStagingModel.png)

By default it will import a CTE and rename the metrics which can be modified accordingly. 

We can now refine the staging models with : 
1. Renaming Columns 
2. Filtering Rows (removing irrelevant/invalid records)
3. Type casting (Converting data types for consistency)
4. Basic Computations (Converting pennies(p) to pounds(£))
5. Basic Date Transformations

Dbt also provides a [Style Guide](https://docs.getdbt.com/best-practices/how-we-style/1-how-we-style-our-dbt-models) for additional support

i.e. the customer staging model then becoems 

~~~~sql 
with 

source as (

    select * from {{ source('jaffle_shop', 'orders') }}

),

renamed as (

    select
        id as order_id,
        user_id as customer_id,
        order_date,
        status as order_status,
        etl_loaded_at

    from source

)

select * from renamed

~~~~

## Testing

We can apply a series of automated checks in our dbt projects. These can be scaled easily and quickly accross the entire project 

There are 2 types of testing in dbt : 
* **Generic Tests** - Re-usable tests built in dbt we can apply columns and models (ensuring column uniquess or is not null)
* **Singular Tests** - Custom tests we can write using SQL query (i.e. ensuring cusotmers total value > 0)

These can be set up in production to provide alerts

---

When you run 

~~~
dbt test
~~~

you can check all the checks are applied accross the models


### Generic Tests

Highly scalable and resuable, can be pplied to many different models and columns (defined in the `.yml` file )

1. UNIQUE - Ensures values in a columns are unique
2. NOT_NULL - No null values 
3. ACCEPTED_VALUES - Checks values belong to specified list 
4. RELATIONSHIPS - validates all values in a column can be referenced to a corresponding (parent table) primary key column (INTEGRITY)

### Singular Test

For specific one off assertions. Set up via `.sql` file stored in the `test` directory 

^ Perfect for validating business logic 

i.e. to look for duplicates we may have the following test 

~~~sql 
select count(*)
  from (select id from test_dim_table)
  group by id
  having count(*) > 1
~~~

If this returns an output we know we have failed the duplicates test and an alert can be configured. 


[Package Hub](hub.getdbt.com) contains a `dbt_utils` package which contains SQL tests that have already been configured with dbt

---

### GENERIC TEST: UNIQUE & NOT NULL 

set up a `.yml` similar to the `_src_` `.yml` but this time we set up a `_stg_` `.yml` for the model we are addressing 

~~~~
models
    ├── marts
    │   └── marketing
    │       └── dim_customers.sql
    └── staging
        ├── jaffle_shop
        │   ├── _src_jaffle_shop.yml
        │   ├── _stg_jaffle_shop.yml*
        │   ├── stg_jaffle_shop__orders.sql
        │   └── stg_jaffle_shop__customers.sql
        └── stripe
            ├── _src_stripe.yml
            └── stg_stripe_payments.sql
~~~~

Note the **`_stg_jaffle_shop.yml`** is new. 

In this `.yml` we define the model, columns and tests accordingly 

~~~~yml 
models:
  - name: stg_jaffle_shop___customers
    columns:
      - name: customer_id
        data_tests:
          - unique 
          - not_null 

  - name: stg_jaffle_shop__orderss
    columns:
      - name: order_id
        data_tests:
        - unique 
        - not_null

~~~~

Note: the **renamed** staging model columns have been used to define the metrics in the `.yml`

To run the testing we use 

~~~bash 
dbt test
~~~

![](/Base/Images/dbtTest.png)

This will go through our `.yml` and if successful returnt he following 

![](/Base/Images/TestUniqueSuccess.png)


---

### GENERIC TEST: ACCEPTED VALUES 

To set up a column to only contain specific values we use the following notation 

 **`_stg_jaffle_shop.yml`** 
~~~~yml
models:
# -name: stg_jaffle_shop__customer
#        ...

  # - name: stg_jaffle_shop__orders
    columns:
      # - name: order_id
          # ...
      - name: order_status
        data_tests:
          - accepted_values:
              arguments:
                values:
                - completed
                - return_pending
                - returned
                - placed
                - shipped

~~~~

OR 

~~~~yml
      - name: order_status
        data_tests:
          - accepted_values:
              arguments:
                values: ['completed','return_pending','returned','placed','shipped']

~~~~

We can specify our tests for specific models that we want to test 

~~~~bash
dbt test --select stg_jaffle_shop__orders
~~~~

This will only run the tests associated with `jaffle_shop__orders` model


---

### GENERIC TEST: RELATIONSHIP TESTS

Use to validate every customer ID key in the `orders` model can be associated with the `customer_id` from `customers` model. We can reference with the following: 

 **`_stg_jaffle_shop.yml`** 
~~~~yml
models:
- name: stg_jaffle_shop__customers
  columns:
    - name: customer_id
      data_tests:
        - not_null
        - unique
#        ...
- name: stg_jaffle_shop__orders
  columns:
      # - name: order_id
      # - name: order_status
      - name: customer_id
        data_tests:
          - relationships:
              arguments:
                to: ref('stg_jaffle_shop__customers')
                field: customer_id

~~~~

Note: dbt update notes now requires usse of `arguements` before the `field` and `to` parameters.

### SINGULAR TESTS

For singular tests remember the `dbt test` command: <br>
! **FAILS** ! if query returns any rows

So our goal is to create a query that **VIOLATES** our understanding. 

---

Reviewing the **stg_stripe_payments** model we can see there are rows for the same `order_id` where 2 succesful payments add to a specific total . 



| payment_id| order_id | payment_method| status| payment_amount |
| :--- | :--- | :--- | :--- | :--- | 
| 1 | 1| credit_card | Success | 10000 | 
| 2 | 2| bank_transfer | Success | 20000 | 
| 3 | 3| coupon| Success | 100 | 
| 4 | 4| bank_transfer | Fail | 1700 | 
| 5 | 4| bank_transfer | Success | 1700 | 
| 6 | 5| bank_transfer | Success | 20000 | 
| 7 | 6| credit_card | Success | 100 | 
| 8 | 7| credit_card | Fail | 1700 | 
| 9 | 7| bank_transfer | Success | 1700 | 
| 10 | 7| bank_transfer | Success | 20000 | 
| 11 | 8| credit_card | Success | 100 | 
| 12 | 9| coupon | Fail | 1700 | 
| 13 | 10| bank_transfer | Success | 1700 | 

For examples `order_id` : ( 7 ,4 )

We can see several transactions wen through to add to a particular total. We want to ensure the sum of all payments for an order is not negative. 

To set up the generic tesst we first need tos et up a `.sql` file for our test. 

in the test directory : 

~~~~
├── models
│   ├── marts/
│   └── staging/
│       ├── jaffle_shop/
│       └── stripe/
└── test/
    └── *assert_stg_stripe__payments_total_positive.sql
~~~~

Ensure the title is clear on what the file will include. 

In the example where we want the sum of amount to be less than 0 we want to test for negative total amounts : 

**assert_stg_stripe__payments_total_positive.sql**
~~~~sql 
select 
    order_id, 
    sum(payment_amount)
from {{ ref('stg_stripe__payments') }}
group by order_id
having sum(payment_amount) < 0 
~~~~

*Note: we renamed `amount` as `payment_amount` in the staging model so will need to referene the renamed metric. 

any models in the test directory should run as they've been referenced in the `dbt_project.yml' using 

~~~yml 
test-paths: ["tests"]
~~~

---

### TESTING SOURCESS 

We can test sources similarly. Review the `_src_` `.yml` files that were set up to configure the dbt sources. 

Recall: **`_scr_jaffle_shop.yml`**
~~~~yml
sources:
  - name: jaffle_shop
    database: raw
    schema: jaffle_shop
    config:
      freshness:
        warn_after:
          count: 21
          period: day
        error_after:
          count: 22
          period: day
      loaded_at_field: etl_loaded_at
    tables:
      - name: customers
        config:
          freshness: null
      - name: orders
~~~~

We can now add similar `unique` and `not_null` tests. 

NOTE: we must reference the column AS IT APPEARS in the raw table. 

~~~~yml
sources:
  - name: jaffle_shop
    database: raw
    schema: jaffle_shop
    # ...
    tables:
      - name: customers
        columns:
          - name: id
            data_tests:
              - not_null
              - unique
        config:
          freshness: null
      - name: orders
        columns:
        - name: id
          data_test:
            - not_null
            - unique
~~~~

REMEMBER: reference the metrics from the original data source. 

Now to test just the ssource : 

~~~~bash 
dbt test --select source:jaffle_shop
~~~~

or to test all our sourcess 

~~~~bash 
dbt test --select source:*
~~~~

---

Testing Data and Model Integrity

* Test sources to validate the quality of raw data 
* Test models to validate the transformations of the business logic applied 

---

#### USING DBT COPILOT TO GENERATE DATA TESTS

First check Account Settings to enable AI and dbt Copilot 

When you select a model you now shoul have the option `dbt Copilot` and `Generic tests`

This will analyse the models and generate the `.yml` code for you

Note : Always remember to inspect and check the work produced. Also review the location of generated code to ensure it's configured in the correct directory 

The dbt function 

~~~~bash 
dbt build
~~~~

is a combination of 

~~~~bash
dbt test
~~~~

AND 

~~~~bash 
dbt run 
~~~~

**BUILD** essentially runs all the tests and runs models in DAG order

--- 

## DBT Build Command

Dbt **BUILD** allows for a layered approach to quality control. 

1. Testing the Sources 
2. First layer of our models (i.e staging models)
3. Immediately tests the staging models associated with the first layer of models
4. Repeats downstream for any models depending on the first layer of staging models . 

If at any point there's a failure the build process stops completely.

---

Other dbt commands 


~~~~bash 
dbt seed
~~~~
Loads CSV data into your warehouse tables 

~~~~bash
dbt snapshot
~~~~
Tracks slowly changing dimensions in your data


---

Clonsider the following Lineage graph : 

![](/Base/Images/dimCustomerLineage.png)


Even when building upstream using 

~~~~bash 
dbt build --select +dim_customers
~~~~

If a test fails for the `stg_jaffle_shop__orders` model dbt will stop the build process as the model has dependency down the lines and `SKIP` all models and tests from `stg_jaffle_shop__orders` 

![](/Base/Images/dbtFailedTest.png)

We can see the test associated with `stg_jaffle_shop__orders` failed and thus we have : 

![](/Base/Images/dbtSkipDependencies.png)


## Documentation

We can document a project at 3 levels: 
1. Model Level -  High level description for table or views
2. Source Level - Document raw source tables 
3. Column Level - Description for each individual field

---

### Writing Documentation 

We can add the documentation in the `.yml` files : 

**`_stg_jaffle_shop.yml`**

~~~~yml
models: 
  - name : stg_jaffle_shop__customers
    description : The grain of this table is one unique customer per row 
    columns:
      - name: customer_id
        description : Primary key for the customers table
        data_tests:
        - unique
        - not_null 
  - name: stg_jaffle_shop__orders
    description : One order per row
    columns: 
    - name: order_id
      description: Primary key for the orders table
      data_tests: 
      - unique
      - not_null 

~~~~

### Writing Doc Blocks

In order to keey our `.yml` file clean we can reference a larger text file (saved as an `.md` file) 

~~~~
models
    ├── marts/
    │   └── ...
    └── staging
        ├── jaffle_shop
        │   ├── *jaffle_shop_docs.md
        │   └── ...
        └── stripe/
~~~~

We would store the information in the md using the `{%docs order_status %}` notation. 

**jaffle_shop_docs.md**
~~~~md
{% docs order_status %}
    
One of the following values: 

| status         | definition                                       |
|----------------|--------------------------------------------------|
| placed         | Order placed, not yet shipped                    |
| shipped        | Order has been shipped, not yet been delivered   |
| completed      | Order has been received by customers             |
| return pending | Customer indicated they want to return this item |
| returned       | Item has been returned                           |

{% enddocs %}

~~~~

To reference this in the `.yml` file we us quote notation 


**`_stg_jaffle_shop.yml`**

~~~~yml
models: 
  - name : stg_jaffle_shop__customers
    # ...
  - name: stg_jaffle_shop__orders
    description : One order per row
    columns : 
    - name : order_id
    - name : ordr_status
      description: "{{ doc('order_status') }}"
      data_tests:
      - accepted_values:
        values: 
        # ...

~~~~

---

It is good best practice to define raw data sources and clarifying ambigious columns and metrics. 

**_src_jaffle_shop.yml**
~~~~yml 

sources:
  - name: jaffle_shop
    database: raw
    schema: jaffle_shop
    description: A clone of a Postgres application database
    # config: ...
    tables:
      - name: customers
        description : Raw customer datga
        columns:
        - name : id
          description: Primary key for our customers data
      - name: orders
        description : Raw orders data
        columns:
          name: id
          description : Primary key for order data

~~~~

Note when documenting any metrics from the source ensure the column names are associated with the raw data columns

---

If using DBT Copilot to generate documentation. Remember to always review the results. 

You can request Dbt Copilot to `generate documentation` this will udpate the description to the `.yml` files associated with the sources and models. 

NOTE: ALWAYS REVIEW THE PROVIDED DESCRIPTIONS PROVIDED BY COPILOT 

Copilot is a good tool for starting off. 

Copilot will also carry over all the column names so there's no need to investigate and add them to the `.yml` one by one 


---

### Merging to Main

Before merging any brances to main it's good practice to perform a final check to ensure the branch is polished. We do this with the following : 

1. Sources are configured `_scr_[project].yml` files 
2. Written descriptions for our sources, models and columns (provided in the `_stg_[project].yml`)
3. these should include `descriptions` and `data_tests`
4. Staging Models refactores with an import CTE (from the source) 
5. Models located in appropriate folder (i.e. `marts/[sub_group_directory]`)
6.  `.yml` files associated with each models (located in the same folder as the model) for added testing and descriptions

Once tested and satisfied, you can `Merge to Main branch` domain

---

## Deployment

We use the main branch for building models. 
Use Development models to test and develop the next set of models. Once Deployed and merged to Main the model should update the next schedueled run. 

---

### Set up a Deployment Environment 

2 Different Type of Environments : 

1. Development Environment Type <br>(you can only have one) 
2. Deployment Environment Type <br>There are different types of deployment types (General, Staging, Production)


You can specify which branch to run projects in the Environment. (Default is main)

We can set up the Connection from the established connections.

We can assign a profile using key value pairs. 

(Review `ConfigureSnowflake2dbt.md` for details on assigning publick rsa keys to profiles to embed the connection)

***NOTE*** : When deploying a real dbt Project, you should set up a separate data warehouse account for this run. This should not be the same account that you personally use in development.

***IMPORTANT*** : The schema used in production should be different from anyone's development schema.

---

### Schedule a job on dbt 

In the example we want to build a daily build on the entire model 

![](/Base/Images/dimCustomerLineage.png)

We can `deploy a job` associated to the `Production` environment

* Select `Run source freshness`
* Specify the models we want to run using dbt commands to build the whole project regularly with the command: 

~~~~bash 
dbt build
~~~~

Execution Settings 

We can schedule the run indicating **Intervals**/**Specific Time** or [**Cron schedule**](https://crontab.guru)

We can also trigger the job using **Job Completion**
(i.e. when a separate job has been successfully completed or failed )

![](/Base/Images/SuccessfulJobRun.png)

We can also review the model Timings : 

![](/Base/Images/ModelTimings.png)

And if necessary we can `rerun` from  the start of the job or a point of failure. 

---

### dbt Catalog 

This allows us to dive into the metadata and details behind the Project. 

Model details, Lineage, Test details and recommendations. We can assess the recommendations that are inbuilt with dbt. 

dbt Catalog also provides metrics on Model Run times and Model usage across the project. 

