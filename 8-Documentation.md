# Documentation

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
