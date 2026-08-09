# Building Models

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

### _MAKE SURE TO SAVE THE FILE!_ 

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
## Overriding configuration parameters

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

### _MAKE SURE TO SAVE THE FILE!_

To run just the model by itself: 
* to the left of Build icon 
* Run model 

**dbt command line code** 
~~~~
dbt run --select my_first_dbt_model
~~~~

---

#### If Lineage is currently unavailable 

Type the name of the projects in the search and Update Graph

---

## Modularity

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

## Staging 

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

![](dbtLineageCustomerModel.png)


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

### Model Naming Conventions 

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

## Cleaning and Reorganizing 

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


## Materialization strategies 

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


## Working 

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

![](dimCustomerLineage.png)


#### Useful Resources 

* [dbt Commants](https://docs.getdbt.com/category/list-of-commands?version=2.0)
* [Syntax Overviews](https://docs.getdbt.com/reference/node-selection/syntax?version=2.0)
* [SQL Query Anatomy](https://medium.com/@lomso.dzingwa/the-anatomy-of-a-sql-query-189dd0664851)
* [Subqueries and CTEs](https://medium.com/@datainsights17/subqueries-and-ctes-in-sql-aa1ff4b17686)






