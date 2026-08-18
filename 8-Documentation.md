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

