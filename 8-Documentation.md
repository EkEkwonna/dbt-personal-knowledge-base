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

### Writing Doc Block 
