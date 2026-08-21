When creating a new Github Repo for dbt a Readme file should not be included as it will cause issues for the project initialisation 

```SYSADMIN``` & ```SECURITYADMIN``` are required to set up databases and roles in Snowflake


# Deploying a Production Environment

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

#### Deployment Credentials

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

### dbt_project.yml 

By Default the dbt_project.yml should look like : 

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