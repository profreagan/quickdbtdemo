# dbt Exercise Instructions #
- Let's jump into a ELT demo using Airbyte and dbt. Remember the insurance exercise where we took an insurance transaction and created a dimensional model based on the given business process and transaction? We decided on the below dimensional model:

![alt text](insurancedimensionalmodel.png)

- Now, we are going to use a semi-normalized transactional database given to us by Sarah's insurance company. Let's work through an ELT process for Sarah's insurance company using Airbyte and dbt. 
### Extract and Load (FiveTran) ###
- Sign into fivetran
- Click on 'Connections'
    - Click 'Add Connection'
- Search for and select 'Amazon RDS for PostgreSQL'
- Select the destination you previously set up for Snowflake
- Set the Destination schema prefix to `insurance`
- Set the Host to `database-1.c3ckkcekkkxp.us-east-1.rds.amazonaws.com`
- Set the user to `fivetran_usr`
- Set the password to `dw_fivetran`
- Set the database to `insurance`
- Set Update Method to 'Detect Changes via Fivetran Teleport Sync'
- Click 'Save & Test'
- Click 'Continue' even if it says 'XMIN extensions not enabled'
- When you get to the Select Data to Sync page, make sure that the following 4 tables are selected and click 'Save & Continue':
    - agents
    - claims
    - customers
    - policies
- Choose to allow all changes
- Click 'Sync Now' in the top right corner
- Wait for the sync to finish, login to Snowflake, check to see if you have a new schema in your database called `insurance_dw_source`
    - Confirm that the tables created and that they have data

### Transform (dbt) ###
- Login to dbt Cloud
- Click Develop > Cloud IDE
- Click Initialize dbt project
    - All of the necessary dbt files and folders will be created
- Click Commit and sync
- Click 'Create New Branch'
    - Name the branch `dbt-exercise`
- Right click on the macros directory and create a new file called `generate_schema_name.sql`. This macro will allow us to use custom schemas when we create models.
    - Copy and paste the following code into the newly created macro file:
```
{% macro generate_schema_name(custom_schema_name, node) -%}

    {%- set default_schema = target.schema -%}
    {%- if custom_schema_name is none -%}

        {{ default_schema }}

    {%- else -%}

        {{ custom_schema_name | trim }}

    {%- endif -%}

{%- endmacro %}
```

- Create a packages.yml file in the same folder as your dbt_project.yml file. Paste this in the packages.yml file:
```
packages:
  - package: dbt-labs/dbt_utils
    version: 1.1.1

  - package: calogica/dbt_date
    version: 0.9.1
```  
- Save the file, after you have done that, you can go to your terminal and type `dbt deps` to install dbt dependencies

- Right click on the models directory and create a new folder inside of it. (Be careful not to create it inside of the example directory.)
- Call this new folder `insurance`
- Right click on insurance and create a new file. Name this file `_src_insurance.yml`
    - In this file we will add all of the sources for the insurance tables
- Below is the code that we will use in this file:
```
version: 2

sources:
  - name: insurance_landing
    database: firstnamelastname
    schema: insurance_dw_source
    tables:
      - name: agents
      - name: claims
      - name: customers
      - name: policies
```


#### dim agent ####
- Create a new file inside of the insurance directory called `dim_agent.sql`
- Here is the code for dim_agent:
```
{{ config(
    materialized = 'table',
    schema = 'dw_insurance'
    )
}}

SELECT
agentid as agent_key,
agentid,
firstname,
lastname,
email,
phone
FROM {{ source('insurance_landing', 'agents') }}
```
- Save the file and run `dbt build --select dim_agent` in the terminal to build the model.

#### dim customer ####
- Create a new file inside of the insurance directory called `dim_customer.sql`
- Here is the code for dim_customer:
```
{{ config(
    materialized = 'table',
    schema = 'dw_insurance'
    )
}}


select
customerid as customer_key,
customerid,
firstname,
lastname,
dob,
address,
city,
state,
zipcode
FROM {{ source('insurance_landing', 'customers') }}
```

- Save the file, after you have done that, you can go to your terminal and type `dbt run -m dim_customer` to build the model.
    - Go to Snowflake to see the newly created table!

#### dim_policy ####
- Create a new file inside of the insurance directory called `dim_policy.sql`
- Here is the code for dim_policy:
```
{{ config(
    materialized = 'table',
    schema = 'dw_insurance'
    )
}}


select
policyid as policy_key,
policyid,
policytype
FROM {{ source('insurance_landing', 'policies') }}
```

- Save the file, after you have done that, you can go to your terminal and type `dbt run -m dim_policy` to build the model.
    - Go to Snowflake to see the newly created table!


#### dim_date ####
- Create a new file inside of the insurance directory called `dim_date.sql`
- Here is the code for dim_date (This will create a date dimensions for us and then we we use a cte to select out of that only the fields we want):


```
{{ config(
    materialized = 'table',
    schema = 'dw_insurance'
    )
}}

with cte_date as (
{{ dbt_date.get_date_dimension("1990-01-01", "2050-12-31") }}
)

SELECT
date_day as date_key,
date_day,
day_of_week,
month_of_year,
month_name,
quarter_of_year,
year_number
from cte_date
```

- The dbt_date macro comes from the below package. Create a packages.yml file in the same folder as your dbt_project.yml file. Paste this in the packages.yml file:
```
  - package: calogica/dbt_date
    version: 0.9.1
```
- Then run `dbt deps`

- Save the file, after you have done that, you can go to your terminal and type `dbt run -m dim_date` to build the model.
    - Go to Snowflake to see the newly created table!


#### fact_claim ####
- Create a new file inside of the insurance directory called `fact_claim.sql`
- Here is the code for fact_claim:
```
{{ config(
    materialized = 'table',
    schema = 'dw_insurance'
) }}

SELECT
    p.policy_key,
    cu.customer_key,
    a.agent_key,
    d.date_key,
    c.ClaimAmount
FROM {{ source('insurance_landing', 'claims') }} c
INNER JOIN {{ source('insurance_landing', 'policies') }} pd ON c.PolicyID = pd.PolicyID
INNER JOIN {{ ref('dim_policy') }} p ON pd.PolicyID = p.policyid 
INNER JOIN {{ ref('dim_customer') }} cu ON pd.CustomerID = cu.customerid 
INNER JOIN {{ ref('dim_agent') }} a ON pd.AgentID = a.agentid 
INNER JOIN {{ ref('dim_date') }} d ON d.date_day = c.ClaimDate

```

- Save the file, after you have done that, you can go to your terminal and type `dbt run -m fact_claim` to build the model.
    - Go to Snowflake to see the newly created table!

- If for some reason you need to run all files then you can run: `dbt run -m insurance`.

#### Model Attributes YAML file ####
- Create a new file inside the insurance directory called `_schema_insurance.yml`
- This file contains metadata about the models you build. It is not required, but highly encouraged by dbt to document your models. It's a lot of tedious work so doesn't seem super necessary for this exercise, but building the file is still a good habit to get into.
- Structure the file just like the following code:
```
version: 2

models:
  - name: dim_agent
    description: "Insurance Agent Dimension"
  - name: dim_customer
    description: "Insurance Customer Dimension"
  - name: dim_date
    description: "Insurance Date Dimension"
  - name: dim_policy
    description: "Insurance Policy Dimension"
  - name: fact_claim
    description: "Insurance Claim Fact"
```

## Create a semantic layer model (time permitting)
- Create a model that can query from the data warehouse we just built and reference upstream models.
- Create a new file called `claims.sql` inside of the insurance directory.

## View Lineage and Generate Docs ##
- View Lineage for your semantic layer model by clicking on the model in the file explorer and clicking lineage on the bottom window.
- Submit a screenshot of the DAG.
- Run `dbt docs generate` in the command line
- Click the docs icon to the right of the `Change branch` link.
- Select the claims model from the project explorer on the left.


## Create a Pull Request on GitHub for the changes you have made ##
- Click Save on any files that you have made changes in.
- Click `Commit and Sync`
- Type a commit message explaining the changes you've made. Click `Commit Changes`.
- Click `Create a pull request on GitHub`
    - You will be redirected to GitHub
- Review your changes and click `Create pull request`
- Type a description about the changes you are proposing to the project.
- Click `Create Pull Request`
- merge your branch into the main branch by clicking `Merge pull request`.
