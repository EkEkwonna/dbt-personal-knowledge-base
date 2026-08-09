# Referencing Sources

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

![](dimCustomerLineage.png)

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

![](srcLineage.png)

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

