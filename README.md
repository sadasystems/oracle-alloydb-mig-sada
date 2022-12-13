This POC covers in detail the migration steps required to migrate an Oracle database to AlloyDB (PostgreSQL) using Striim - A third party solution that has a validated integration with AlloyDB.

## Overview

Database migration using Striim has a two-step approach:

Step-1: Full one-time, initial replication of Oracle database dump.

Step-2: Continuous replication of every transaction at Oracle through CDC (Change Data Capture)

The migration is facilitated by installing the Striim software on a separate server. Within GCP, the Striim server can be created in one of two ways:

- Option-1 :- Use a pre-configured Google compute engine instance from the GCP marketplace.

- Option-2 :- Use a custom Google compute instance engine instance.
    
Both options require a license that must be purchased separately.

>For this POC, we chose Option-2.

For this POC, the following GCP services are required:
- A project to host the GCP services.
- VPC to host the Oracle and the Striim GCE instance.
- An AlloyDB cluster with a single instance.
- Firewall rules for allowing connectivity between the Oracle database server, the Striim server and the AlloyDB instance.

The Oracle database and AlloyDB single instance cluster were set up using using the respective vendor documentation.The Striim server establishes connectivity to Oracle and AlloyDB databases through its built-in adapter in the Striim application or pipeline through the flow designer drag-and-drop interface.

We utilized the following Striim adapters for the POC:

-   Database Reader: Reads data from Oracle source database during initial load.
-   Oracle Reader: Read data using LogMiner from the oracle database during the CDC replication stage
-   Database Writer: Writes data to AlloyDB during initial load and CDC replication.

## Configuration of Oracle & AlloyDB Instance

-   The following table summarizes the Oracle database, Alloy DB database and Striim server configuration

 ||Oracle DB|AlloyDB Instance|Striim Server|
| :- | :- | :- | :- |
|vCPU|4|4|4|
|Memory|32 GB|32 GB|15 GB|
|Data disk|SSD - 300 GB|Default|30 GB|
|OS|CentOS 7|Default|CentOS 7|
|Cluster/Database Names|Standalone - LQDB|Primary Instance - oracle2postgres| N/A |
|GCE instance name|instance-oradb | N/A |  instance-striim|


> **NOTE:** In Oracle database we are going to use the existing schema **HR** for testing.

- Firewall: the following ports must be open for communication with Striim
    -   Source Oracle database: port tcp/1521
    -   Striim server running the web UI: port tcp/9080 (http) and/or tcp/9081 (https)
    -   Target AlloyDB instance: port tcp/5432

## Implmentation Guide:
- [Step-1](#1-connectivity-oracle-striim-alloydb) Connectivity ( Oracle <-> Striim <-> AlloyDB)
- [Step-2](#2-preparing-source-database---oracle) Preparing the source Oracle database
- [Step-3](#3-preparing-striim-gce-instance) Preparing Striim GCE instance
- [Step-4](#4-schema-conversion-to-alloydb) Schema conversion to Alloy DB
- [Step-5](#5-preparing-target-database---alloydb) Preparing target AlloyDB database 
- [Step-6](#6-initial-oracle-database-load-to-alloydb) Initial Oracle database load to AlloyDB 
- [Step-7](#7-continuously-replication-cdc---oracle-to-alloydb) Continuously replication CDC - Oracle to AlloyDB 
- [Step-8](#8-promote-the-target-alloydb-database-cut-over) Promote the Target AlloyDB database (cut-over) 


## 1. Connectivity ( Oracle <-> Striim <-> AlloyDB)

### Connection to Oracle Instance.

Make sure you can make the successful sqlplus connection hr schema on
oracle database.
```sql
 $> sqlplus hr/xx@LQDB
```

### Connection to AlloyDB Cluster instance

Enable Project access to AlloyDB : 
[https://cloud.google.com/alloydb/docs/project-enable-access](https://cloud.google.com/alloydb/docs/project-enable-access)

Connection to alloyDB instance "oracle2postgres" through its private IP using psql client.

The default USERNAME is "postgres" and the password you used while creating the cluster for PASSWORD.

```sql
$> psql -h xx.xx.xx.xx -U postgres
```
### Connectivity between Striim, Oracle and AlloyDB

Make sure firewall rules are not blocking the

-   Striim application to Oracle DB
    -   By default, the Oracle Listener is configured on port 1521
    -   Check \$ORACLE_HOME/network/admin/tnsnames.ora

-   Striim application to Alloy DB
    -   By default AlloyDB(Postgresql) listens on port 5432

a.  Connectivity Test through telnet
    i.  From instance-striim to instance-oracle
```
[saurabh_patel@instance-striim ~]$ telnet xx.xx.xx.xx 1521
Trying xx.xx.xx.xx...
Connected to xx.xx.xx.xx.

```
ii. From instance-oracle to instance-striim
```
[saurabh_patel@instance-oradb ~]$ telnet xx.xx.xx.xx 22
Trying xx.xx.xx.xx...
Connected to xx.xx.xx.xx.
```

iii. From Instance-striim to AlloyDB. AlloyDB listens on port 5432

```
saurabh_patel@instance-striim ~]$ telnet xx.xx.xx.xx 5432
Trying xx.xx.xx.xx...
Connected to xx.xx.xx.xx.
```

## 2. Preparing source database - Oracle

While there are different Oracle CDC sources, in this POC we are going to use
[LogMiner](https://docs.oracle.com/en/database/oracle/oracle-database/19/sutil/oracle-logminer-utility.html#GUID-3417B738-374C-4EE3-B15C-3A66E01AE2B5).
You can read about alternate options in [Alternate Oracle CDC
sources](https://cloud.google.com/architecture/migrate-oracle-to-cloudsql-for-postgresql-using-striim#alternate_oracle_cdc_sources).

Prepare the Oracle database:- [https://www.striim.com/docs/en/basic-oracle-configuration-tasks.html](https://www.striim.com/docs/en/basic-oracle-configuration-tasks.html)

- a)  Enable Striim's archivelog
    ```sql
    SQL\> select log_mode from v\$database;
    NOARCHIVELOG

    SQL\> shutdown immediate;
    SQL\> startup mount
    SQL\> alter database archivelog;
    SQL\> alter database open;

    SQL\> select log_mode from v\$database;
    ```
- b)  Enable Striim supplemental log data and Primary Key logging

    ```sql
        SQL> select supplemental_log_data_min, supplemental_log_data_pk from v$database;
        SUPPLEME SUP
        -------- ---
        NO       NO
        
        SQL> ALTER DATABASE ADD SUPPLEMENTAL LOG DATA;
        Database altered.
        SQL> ALTER DATABASE ADD SUPPLEMENTAL LOG DATA (PRIMARY KEY) COLUMNS;
        Database altered.
        
        SQL> select supplemental_log_data_min, supplemental_log_data_pk from v$database;
        SUPPLEME SUP
        -------- ---
        YES      YES
        SQL> 
    ```

- c)  Create an Oracle user with LogMiner Privileges for Striim


    To run these steps

    -  In case of CDB database, you must be connected to the container database (CDB), regardless of whether you\'re migrating a CDB or a pluggable database (PDB).

    - In our case, it's NON-CDB database
    ```sql
    create role striim_privs;
    grant create session,
    execute_catalog_role,
    select any transaction,
    select any dictionary
    to striim_privs;
    grant select on SYSTEM.LOGMNR_COL$ to striim_privs;
    grant select on SYSTEM.LOGMNR_OBJ$ to striim_privs;
    grant select on SYSTEM.LOGMNR_USER$ to striim_privs;
    grant select on SYSTEM.LOGMNR_UID$ to striim_privs;
    create user striim identified by striim default tablespace users;
    grant striim_privs to striim;
    alter user striim quota unlimited on users;

    grant LOGMINING to striim_privs;
    ```


- d)  Create a Striim quiescemarker table

    Striim\'s Oracle Reader adapter for CDC needs a table for storing metadata when it quiesces an application. If you use LogMiner as a source for CDC (in our case yes), then you need the quiescemarker table.

    In case of CDB, you must be connected to the CDB when following the steps to create the table.

    ```sql
    SQL> conn hr/xx@LQDB
    Connected.
    SQL> 
    SQL> CREATE TABLE QUIESCEMARKER (source varchar2(100), 
    status varchar2(100),
    sequence NUMBER(10),
    inittime timestamp, 
    updatetime timestamp default sysdate, 
    approvedtime timestamp, 
    reason varchar2(100), 
    CONSTRAINT quiesce_marker_pk PRIMARY KEY (source, sequence));  2    3    4    5    6    7    8  

    Table created.

    SQL> 
    ```


- e)  Fetch the SCN ( system change number) for the Oracle database.

    On the Oracle database, get the oldest SCN:
    ```sql
    SQL\> SELECT MIN(start_scn) FROM gv\$transaction;
    ```

    To get the current SCN from database;
    ```sql
    > SQL\> Select CURRENT_SCN from v\$database;
    ```


    Capture this number. You\'ll need it later in the CDC replication
    pipeline steps.

## 3. Preparing Striim GCE instance

Perform the following steps on each Striim server that runs an Oracle
Reader adapter:

1)  Striim Setup on compute engine (VM)

### It's going to Install Striim (4.1.0.2) and Oracle Java (1.8) SDK 64-bit

a.  Login to Striim VM and install Git.
```sh
    sudo yum update -y
    sudo yum install git -y
```
b.  Change to root user by command 
```sh
    sudo su -
```
c.  Export license key, product key, company name, total memory, and cluster name as an environment variable.
``` sh
    export company_name=SADA
    export licence_key=xx-xx-xx-xx-xx-xx
    export product_key=xx-xx-xx
    export total_memory=16
    export cluster_name=Striim
```
d.  Clone the repository in the home directory
```sh
    git clone https://github.com/sadasystems/oracle-alloydb-mig-sada.git
```
e.  Change directory to 
```sh
    cd oracle-alloydb-mig-sada/install/ .
```
f.  Execute striim-install.sh script 
```sh
    ./striim-install.sh

    Please answer the following to get started with the installation process.
Which operating system are you using? (amazon, centos, redhat, ubuntu or debian)
    centos

 Install Striim Version 4.1.0.1 
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 5628k  100 5628k    0     0  5608k      0  0:00:01  0:00:01 --:--:-- 5611k
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 1650M  100 1650M    0     0  42.7M      0  0:00:38  0:00:38 --:--:-- 48.4M
Preparing...                          ################################# [100%]
Updating / installing...
   1:striim-dbms-4.1.0.1-1            ################################# [100%]
..
..
..
..
 Install Java JDK 1.8 
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  141M  100  141M    0     0  42.5M      0  0:00:03  0:00:03 --:--:-- 42.5M
```
g.  After the script installs Java and Striim, it will show a prompt for
   > **NOTE** Password = striim ( for keystore,sys, admin) and 
   > You will login to Striim console with the admin credentials you entered in this step.

```
Setup Striim Credentials 
Please enter the KeyStore password: ******
Creating the KeyStore.
Please enter the sys user password: ******
Saving the sys user password in the KeyStore.
Please enter the admin user password: ******
Saving the admin user password in the KeyStore.
MDR Types: [1: Derby] [2: Oracle] [3: Postgres] 
Please enter the MDR type: 1
Saving the default Derby MDR password in the KeyStore.

 Successfully updated startup.properties file 
Failed to execute operation: File exists
Failed to execute operation: File exists
 Succesfully started Striim node and dbms 

Would you like to create Initial Load application(s)? (yes or no)
    yes

 
Enter source JDBC URL: 
    jdbc:oracle:thin:hr/xx@xx.xx.xx.xx:1521:LQDB
 
Enter schemas and tables to exclude:
 MDSYS;OUTLN;CTXSYS;SYSTEM;DVSYS;DBSNMP;GSMADMIN_INTERNAL;OJVMSYS;ORDSYS;XDB;ORDDATA;SYS;WMSYS;LBACSYS
Enter username: striim
Enter password: striim
 
Enter # IL applications: 1
      Note: We recommend entering a value of 1 for the initial PoC 
      but if you'd like to increase the performance of the initial load, 
      enter a number greater than one and a Java script will calculate the size of your tables and split the applications accordingly. 
Enter the # Striim cores: 4
    Note: This value is used to parallelize the initial load and split the threads depending on the number of applications you created.
 
Enter target JDBC URL: 
      jdbc:postgresql://xx.xx.xx.xx:5432/hr
Enter username: postgres
Enter password: postgres
Enter application name: oraalloydb
Enter source name: ora
Enter stream name: orastream
Enter target name:  alloydb
 
TQLs generated successfully.
Current node started in cluster : Striim, with Metadata Repository 
Registered to: SADA
ProductKey: xx-xx-xx
License Key: xx-xx-xx-xx-xx-xx
License expires in 30 days 12 hours 56 minutes 53 seconds
Servers in cluster: 
  [this] Sxx_xx_xx_xx [8ef147e0-47bc-4e9d-a815-1b6aae6dc26f] 
 
started.
Please go to http://xx.xx.xx.xx:9080 or https://xx.xx.xx.xx:9081 to administer, or use console


```

> **NOTE** TQL File location = oracle-alloydb-mig-sada/install/oraalloydb_1.tql

- Created the windows VM to access the striim application console.

    - Create the windows VM
    - Enable the Display option on VM

    -   Connect through IAP tunnel
    ```sh
    -   gcloud compute start-iap-tunnel striim-console 3389 \--local-host-port=localhost:3389 --project "PROJECT_NAME"
    ```
    > **NOTE: Make sure Firewall rules are not blcoked to allow connection between
    > striim-console (windows vm) and instance-striim ( striim application)

- Accessing Striim console
    - User name = admin
    - Password = striim

        ![](images/sada-ora-striim-alloydb-poc.001.png)


## 4. Schema conversion to AlloyDB 

Striim typically requires that the target database contains
corresponding tables with the correct schema. Striim only transfers data
between source and target tables.

1.  Schema conversion utility from Striim
2.  open source utility like [ora2pg](https://cloud.google.com/community/tutorials/migrate-oracle-postgres-using-ora2pg)
    - ora2pg used for exporting more type of Oracle database objects (tables,views,sequences,indexes,functions,triggers,procedures,packages,materialized view,grants etc..)


- In this POC we are going to use the Striim utility for schema conversion.

    - open SSH connection to Striim instance.

    ```sh
    [saurabh_patel@instance-striim striim]$ sudo su -

    [root@instance-striim ~]# cd /opt/striim/
    [root@instance-striim striim]# 

    ```
    - To convert the schema.

    ```sh
    bin/schemaConversionUtility.sh \
    -s=oracle \
    -d="jdbc:oracle:thin:@xx.xx.xx.xx:1521:LQDB" \
    -u=hr \
    -p=hr \
    -b="HR.REGIONS;HR.COUNTRIES;HR.LOCATIONS;HR.DEPARTMENTS;HR.JOBS;HR.EMPLOYEES;HR.JOB_HISTORY;HR.QUIESCEMARKER" \
    -t=postgres \
    -f=false
    ```
    - It lists out compatible/incompatible/Foreignkey etc. details

    ```sh
    SCHEMA CONVERSION RESULTS -
        Schema name - HR
        Number of compatible tables - 8
        Number of tables compatible with Striim Intelligence - 0
        Number of incompatible tables - 0
        Number of successful foreign keys - 10
        Number of failed foreign keys - 0
        The resultant output SQL files, and the report of the schema conversion are located at the folder: /opt/striim/oracle-postgres-2022_11_21_16_49_40
        [root@instance-striim striim]#
    ```


- Details verbose schema conversion report is stored in the file conversion_report.txt
    ```sh
    [root@instance-striim oracle-postgres-2022_11_21_16_49_40\]# cat
    /opt/striim/oracle-postgres-2022_11_21_16_49_40/conversion_report.txt
    ```

- With the above outcomes, 
    - we are going to use **"converted_tables.sql"** for table creation and then one time FULL dump is loaded into the destination tables.
    - Then we use **"converted_foreignkey.sql"** to create the foreign key on target.

## 5. Preparing target database - AlloyDB

Connect to the AlloyDB PostgreSQL database 

```sh
    psql -h xx.xx.xx.xx -d hr -U postgres
```
- Create database "hr" and execute the objects creation script
```sql
    Create database hr;
```
1.  Create Schema and execute the script **"converted_tables.sql"** located at </opt/striim/oracle-postgres-2022_11_21_16_49_40\>

```sql
hr=> create schema hr;
CREATE SCHEMA
hr=> 
hr=> 
hr=> \i converted_tables.sql
CREATE TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
hr=> 
hr=> 
To add the new schema to the search path, you use the following command:
hr=> SET search_path TO hr, public;
SET
hr=> SHOW search_path;
 search_path 
-------------
 hr, public
(1 row)

hr=>

```

2.  [Create a checkpointing table](https://www.striim.com/docs/en/creating-the-checkpoint-table.html)
    on the alloydb database:
    - Striim needs this table to maintain checkpoints during the continuous replication process ( When the recovery option is enabled.

```sql
CREATE TABLE hr.chkpoint (
id character varying(100) primary key,
sourceposition bytea,
pendingddl numeric(1),
ddl text);

hr=> \dt
             List of relations
 Schema |     Name      | Type  |  Owner   
--------+---------------+-------+----------
 hr     | chkpoint      | table | postgres
 hr     | countries     | table | postgres
 hr     | departments   | table | postgres
 hr     | employees     | table | postgres
 hr     | job_history   | table | postgres
 hr     | jobs          | table | postgres
 hr     | locations     | table | postgres
 hr     | quiescemarker | table | postgres
 hr     | regions       | table | postgres
(9 rows)

hr=> 

```

## 6. Initial Oracle database load to AlloyDB

This section describes the one-time, initial replication of the Oracle
database to the AlloyDB database.

A.  Created a new app in Striim from a TQL file.

- [Enabled SSH on Windows striim-console](https://cloud.google.com/compute/docs/connect/windows-ssh#running_vm)

- Copied **oraalloydb_1.tql** file to striim-console using [gcloud compute scp](https://cloud.google.com/sdk/gcloud/reference/compute/scp)

```sh
gcloud compute scp oraalloydb_1.tql striim-console:/C:/striim/oraalloydb_1.tql\`
```
B.  Deploy migration pipeline

- Deploy the applicaion
    - It will performance validation e.g. connection,tables etc..
- Start the application.

![](images/sada-ora-striim-alloydb-poc.002.png)

![](images/sada-ora-striim-alloydb-poc.003.png)

![](images/sada-ora-striim-alloydb-poc.004.png)

C.  Enabling foreign key constraints on the AlloyDB table

```sql
hr=>\i converted_foreignkey.sql
```
D.  Test the Data replication and Validation
- On Target fetch the record count
```sql
hr=> select count(1) from hr.employees;
 count
-------
   107
(1 row)
hr=>

```
> **NOTE** For reference CDC TQL File = **install/ora_alloydb_initialload_sample.tql**

## 7. Continuously replication CDC - Oracle to AlloyDB

After the initial load we need to create a separate pipeline to replicate changes to Alloy DB from the source Oracle database. We will be using the SCN during the initial load setup.

-   Once the initial load is completed
    - we start the CDC Application with the SCN and it starts replicating ongoing changes to target the AlloyDB database.

A.  Establish a connection to Oracle from Striim

For continued replication we are going to use the [Striim Oracle Reader adapter](https://www.striim.com/docs/en/oracle-reader-properties.html) to connect from Striim to the Oracle database. This shall capture CDC data from Oracle.

a.  Create the New TQL File "ora_alloydb_cdc.tql" on windows VM **striim-console** with the following content. 
    
-  Location \<C:\\striim\\ora_alloydb_cdc.tql\>
```sh
CREATE APPLICATION ora_alloydb_cdc;

CREATE OR REPLACE SOURCE ora_cdc USING Global.OracleReader (
  Password: 'striim',
  OnlineCatalog:true,
  FetchSize: 10000,
  Tables: 'HR.REGIONS;HR.LOCATIONS;HR.JOB_HISTORY;HR.JOBS;HR.EMPLOYEES;HR.DEPARTMENTS;',
  ConnectionURL: 'jdbc:oracle:thin:@xx.xx.xx.xx:1521:LQDB',
  Username: 'striim' )
OUTPUT TO alloystream_cdc;

CREATE OR REPLACE TARGET alloydb_cdc USING Global.DatabaseWriter (
  ConnectionRetryPolicy: 'retryInterval=30, maxRetries=3',
  BatchPolicy: 'EventCount:10000,Interval:60',
  CommitPolicy: 'EventCount:10000,Interval:60',
  CheckPointTable: 'CHKPOINT',
  StatementCacheSize: '50',
  Tables: 'HR.%,HR.%;',
  DatabaseProviderType: 'Default',
  Password: 'postgres',
  ConnectionURL: 'jdbc:postgresql://10.59.112.2:5432/hr',
  PreserveSourceTransactionBoundary: 'false',
  Username: 'postgres',
  adapterName: 'DatabaseWriter' )
INPUT FROM alloystream_cdc;

END APPLICATION ora_alloydb_cdc;

```
b.  Import **ora_alloydb_cdc.tql** file and create a new app in Striim

![](images/sada-ora-striim-alloydb-poc.005.png)

c.  **Test connection:** Click **Test connection**. The connection URL, username, and password are required to test database connectivity.

> If Striim is successfully able to establish a connection, a green check mark appears.

![](images/sada-ora-striim-alloydb-poc.006.png)

d.  Update the "START SCN" value in the Advance settings.

- This value was captured during the initial load.

e.  TQL File also creates the AlloyDB database writer adapter to the pipeline.

- "Alloydb_cdc" is used for writing data to alloydb.
- Adapter: DatabaseWriter


B.  Enabling recovery and encryption

> It's a best practice to enable recovery so if the striim application
> or the VM goes down then enabling recovery helps ensure that striim
> can continue processing. Striim coordinates the two checkpoints to
> ensure no data were lost or duplicated.

- In the Striim Flow Designer, Select App Settings.
- Enable the recovery option.
- Add the recovery interval - 5 seconds.

You can also enable the encryption which will encrypt all streams of
data movement between the Source and Target.

![](images/sada-ora-striim-alloydb-poc.007.png)

C.  Enable logging exceptions

It's recommended to enable the [exception store](https://www.striim.com/docs/en/create-exceptionstore.html) in Striim. As part of the CDC application, there might be duplicates written by the initial load application. Striim application ignores those errors and writes them to store and continues processing.

- In Striim Flow Designer, Select the Exception icon.
- Click on turn on.

![](images/sada-ora-striim-alloydb-poc.008.png)

D.  Deploy the pipeline

Once the pipeline is ready, the next step is to deploy and start running
the application to synch the oracle tables with the AlloyDB database.

Click on DeployApp Option.

![](images/sada-ora-striim-alloydb-poc.009.png)

Once the application is deployed, click on "Start App"

![](images/sada-ora-striim-alloydb-poc.010.png)
![](images/sada-ora-striim-alloydb-poc.011.png)

E.  Test the Data replication and Validation

At source - Oracle DB

![](images/sada-ora-striim-alloydb-poc.012.png)

At target - Alloy DB

![](images/sada-ora-striim-alloydb-poc.013.png)

![](images/sada-ora-striim-alloydb-poc.014.png)

> **NOTE** For reference CDC TQL File = **install/ora_alloydb_cdc_sample.tql**
## 8. Promote the Target AlloyDB database (cut-over) 

To cut-over to AlloyDB.

- Stop the application
- Make sure lag/latency shall be zero.
    - In Striim's case, total input and output shall match.
    - Check the Striim Monitoring Dashboard for details statistics.
- Once it synched
    - Stop the application
    - Undeploy the application
- Point Application to AlloyDB.