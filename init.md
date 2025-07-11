-- update .env

--sh
openssl genrsa 2048 | openssl pkcs8 -topk8 -v2 des3 -inform PEM -out rsa_key.p8
openssl rsa -in rsa_key.p8 -pubout -out rsa_key.pub

virtualenv venv
source venv/bin/activate
source .env

pip install dbt-snowflake

--SQL
ALTER USER <user> SET RSA_PUBLIC_KEY='MIIBIjANBgkqh...';
CREATE DATABASE JAFFLE_SHOP_CLASSIC_DEV;
CREATE ROLE IF NOT EXISTS JAFFLE_SHOP_CLASSIC_ADMIN;
GRANT ROLE JAFFLE_SHOP_CLASSIC_ADMIN TO ROLE SYSADMIN;
GRANT ALL ON DATABASE JAFFLE_SHOP_CLASSIC_DEV TO ROLE JAFFLE_SHOP_CLASSIC_ADMIN;
GRANT OWNERSHIP ON DATABASE JAFFLE_SHOP_CLASSIC_DEV TO ROLE JAFFLE_SHOP_CLASSIC_ADMIN;
CREATE ROLE IF NOT EXISTS JAFFLE_SHOP_CLASSIC_DBT;
GRANT ROLE JAFFLE_SHOP_CLASSIC_DBT TO ROLE JAFFLE_SHOP_CLASSIC_ADMIN;
GRANT USAGE ON DATABASE JAFFLE_SHOP_CLASSIC_DEV TO ROLE JAFFLE_SHOP_CLASSIC_DBT;
GRANT CREATE SCHEMA ON DATABASE JAFFLE_SHOP_CLASSIC_DEV TO ROLE JAFFLE_SHOP_CLASSIC_DBT;

CREATE WAREHOUSE IF NOT EXISTS JAFFLE_SHOP_CLASSIC_DBT_XSMALL
    WAREHOUSE_SIZE = 'XSMALL'
    AUTO_SUSPEND = 30
    AUTO_RESUME = TRUE
    MIN_CLUSTER_COUNT = 1
    MAX_CLUSTER_COUNT = 10
    SCALING_POLICY = STANDARD
    INITIALLY_SUSPENDED = TRUE;

GRANT USAGE, OPERATE, MONITOR ON WAREHOUSE JAFFLE_SHOP_CLASSIC_DBT_XSMALL TO ROLE JAFFLE_SHOP_CLASSIC_DBT;
GRANT CREATE INTEGRATION ON ACCOUNT TO ROLE JAFFLE_SHOP_CLASSIC_DBT;



USE ROLE JAFFLE_SHOP_CLASSIC_DBT;
USE SECONDARY ROLE NONE;
USE DATABASE JAFFLE_SHOP_CLASSIC_DEV;
CREATE SCHEMA IF NOT EXISTS JAFFLE_SHOP WITH MANAGED ACCESS;
CREATE SCHEMA IF NOT EXISTS INTEGRATIONS WITH MANAGED ACCESS;

-- based on https://docs.snowflake.com/en/user-guide/data-engineering/dbt-projects-on-snowflake-getting-started-tutorial

USE SCHEMA JAFFLE_SHOP_CLASSIC_DEV.INTEGRATIONS;
CREATE OR REPLACE SECRET JAFFLE_SHOP_CLASSIC_DEV.INTEGRATIONS.DBT_GIT_SECRET
  TYPE = password
  USERNAME = 'MichalRogalskiPL'
  PASSWORD = '...';

CREATE OR REPLACE API INTEGRATION DBT_GIT_API_INTEGRATION
  API_PROVIDER = git_https_api
  API_ALLOWED_PREFIXES = ('https://github.com/...')
  ALLOWED_AUTHENTICATION_SECRETS = (JAFFLE_SHOP_CLASSIC_DEV.INTEGRATIONS.DBT_GIT_SECRET)
  ENABLED = TRUE;


-- Create NETWORK RULE for external access integration
CREATE OR REPLACE NETWORK RULE DBT_NETWORK_RULE
  MODE = EGRESS
  TYPE = HOST_PORT
  -- Minimal URL allowlist that is required for dbt deps
  VALUE_LIST = (
    'hub.getdbt.com',
    'codeload.github.com'
    );

-- Create EXTERNAL ACCESS INTEGRATION for dbt access to external dbt package locations
CREATE OR REPLACE EXTERNAL ACCESS INTEGRATION DBT_EXTERNAL_ACCESS
  ALLOWED_NETWORK_RULES = (DBT_NETWORK_RULE)
  ENABLED = TRUE;


-- bash
dbt seed
dbt run