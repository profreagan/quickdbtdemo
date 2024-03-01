# dbt Exercise Instructions #

### Extract and Load (Airbyte) ###
- Ensure that you are connected to the University VPN
- Open the Docker application
- Start Airbyte by opening a terminal and running the following (you may be able to just click the local host link below instead of running the following):
``` cd airbyte ```
``` ./run-ab-platform.sh ```
- Open up a browser and go to http://localhost:8000. It can take a while for the Airbyte service to start, so don't be surprised if it takes ~10 minutes.
    - Username: airbyte
    - Password: password
- Click `Set up a new source`
- When defining a source, select `Microsoft SQL Server (MSSQL)`
    - Host: `stairway.usu.edu`
    - Port: `1433`
    - Database: `5360_insurance`
    - Username: `5360_student`
    - Password: `datawarehousing` (you'll need to click the dropdown for option fields)
- Select `Scan Changes with User Defined Cursor`
- Click `Set up source`
    - Airbyte will run a connection test on the source to make sure it is set up properly
- Create a schema in your firstnamelastname database named `Insurance` and ensure you have a data warehouse named `lastname_wh`

- Once Airbyte has run the connection test successfully, you will pick a destination, select `Pick a destination`.
- Find and click on `Snowflake`
    - Host: `https://rbb67081.snowflakecomputing.com` 
    - Role: `TRAINING_ROLE` 
    - Warehouse: `lastname_WH` 
    - Database: `firstnamelastname` 
    - Schema: `INSURANCE` (create this schema in your firstnamelastname database)
    - Username: 
    - Authorization Method: `Username and Password`
    - Password: 
    - Click `Set up destination`
- Once the connection test passes, it will pull up the new connection window
    - Change schedule type to `Manual`
    - Under `Activate the streams you want to sync`, click the button next to each table.
    - Click Set up connection
    - Click `Sync now`
    - Once it's done, go to Snowflake and verify that you see data in the landing database

### Transform (dbt) ###
- Open VSCode
- File > Open > Select your project (lastname_DW)
- On the top bar of the application, select Terminal > New Terminal
    - This will open a terminal in the directory of your project within VSCode
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
    schema: insurance
    tables:
      - name: agents
      - name: claims
      - name: customers
      - name: policies
```

- If you need to make any changes to your Snowflake information in your dbt project you can change it by going to your dbt profile.yml file. You may need to change the schema. 
    - On a mac, this is located under your user directory. You have to click Shift + command + . in order to see hidden folders. The .dbt folder will appear and inside is profiles.yml
    - On Windows, it's just in the user directory under the .dbt folder and the profiles.yml is inside.
    - Once you have found the profiles.yml file you can open in a text editor, change the needed parameters and save the file. 


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
firstname,
lastname,
hiredate,
branchoffice,
licensenum,
salary
FROM {{ source('insurance_landing', 'agents') }}
```

- Save the file, after you have done that, you can go to your terminal and type `dbt run -m dim_agent` to build the model.
    - Go to Snowflake to see the newly created table!

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
firstname,
lastname,
ssn,
dob,
maritalstatus,
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
policynum as policy_key,
type,
coverage
FROM {{ source('insurance_landing', 'policies') }}
```

- Save the file, after you have done that, you can go to your terminal and type `dbt run -m dim_policy` to build the model.
    - Go to Snowflake to see the newly created table!

#### dim_date ####
- This is another things that I think will make the students excited about dbt!
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
day_of_week,
month_of_year,
month_name,
quarter_of_year,
year_number
from cte_date
```

- The dbt_date macro comes from the below package. Create a packages.yml file in the same folder as your dbt_project.yml file. Paste this in the packages.yml file:

packages:
  - package: calogica/dbt_date
    version: [">=0.9.0", "<0.10.0"]
    # <see https://github.com/calogica/dbt-date/releases/latest> for the latest version tag

- Then run "dbt deps". 

- Save the file, after you have done that, you can go to your terminal and type `dbt run -m dim_date` to build the model.
    - Go to Snowflake to see the newly created table!


#### fact_claim ####
- Create a new file inside of the insurance directory called `fact_claim.sql`
- Here is the code for fact_claim:
```
{{ config(
    materialized = 'table',
    schema = 'dw_insurance'
    )
}}

SELECT
CAST(customerid as TEXT) as customer_key,
CAST(policyid as TEXT) as policy_key,
CAST(agentid as TEXT) as agent_key,
CAST(dateofclaim as TEXT) as date_key,
claimamount
FROM {{ source('insurance_landing', 'claims') }}
```

- Save the file, after you have done that, you can go to your terminal and type `dbt run -m fact_claim` to build the model.
    - Go to Snowflake to see the newly created table!

- If for some reason you need to run all files then you can run: `dbt run -m insurance`.

#### schema yaml file ####
- Create a new file inside the insurance directory called `_schema_insurance.yml`
- This file contains metadata about the models you build. It is not required, but highly encouraged by dbt to document your models. At CHG and Pluralsight we even at column level descriptions here so they persist in Snowflake and our data catalog. It's a lot of tedious work so doesn't seem super necessary for this exericese. But building the file is still a good habit to get into.
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

## Create a semantic layer model (time permitting, or extra credit?)
- Create a model that can query from the data warehouse we just built and reference upstream models.
- Create a new file called `sem_claims.sql` inside of the insurance directory.
- Past the following code (This model uses ref instead of source as in the from statements. This will allow dbt to build lineage dag of the model dependencies.):
```
{{ config(
    materialized = 'table',
    schema = 'dw_insurance'
    )
}}

SELECT
c.firstname as customer_first_name,
c.lastname as customer_last_name,
c.ssn as customer_ssn,
c.dob as customer_dob,
c.maritalstatus as customer_marital_status,
c.zipcode as customer_zipcode,
a.firstname as agent_first_name,
a.lastname as agent_last_name,
a.branchoffice as agent_branch_office,
p.policynum as policy_number,
p.type as policy_type,
p.coverage as policy_coverage,
d.date_day as date_of_claim,
f.claimamount
FROM {{ ref('fact_claim') }} f

LEFT JOIN {{ ref('dim_agent') }} a
ON f.agent_key = a.agent_key

LEFT JOIN {{ ref('dim_customer') }} c
ON f.customer_key = c.customer_key

LEFT JOIN {{ ref('dim_policy') }} p
ON f.policy_key = p.policy_key

LEFT JOIN {{ ref('dim_date') }} d
on f.date_key = d.date_key
```

- In order to view lineage, the dbt power user extension must be installed. Click on the Lineage tab in vscode (down by the terminal on the bottom), if you are inside the sem_claims.sql model, you should be able to see lineage for that model.