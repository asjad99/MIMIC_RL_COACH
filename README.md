### ABOUT 

This experiment was done to demonstrate how batch RL for Decision Support for clinicians 

We cover teh following 


- Data Acquisition 
- Pre-Processing 
- Exploring 
- Feature Encoding
- Modeling 
----------------------------------------------------
----------------------------------------------------

### Data Acquisition 


#### 1. Request access to acquire the data:

Create an account https://physionet.org/ here and follow instruction to request access to the data. https://physionet.org/content/mimiciii/1.4/

Meantime view the scheme here: schema: https://mimic.physionet.org/mimicdata/schema/

and read details about how the dataset was constructed here: 


#### 2. Launch an instance: 
For folks performing data analysis without access to a full team of data engineers that can build data pipelines, its usefull to be able to quickly access 


https://medium.com/@junseopark/from-zero-to-aws-ec2-for-data-science-62e7a22d4579


to get to this aws console:


#### 3. I use transmit and iterm simultaneously


![](https://paper-attachments.dropbox.com/s_9C453E0E6A99C23A853A12A22A94087A14951068B3D837D1A2BDE0BFBFDEAD7A_1594289590617_Screen+Shot+2020-07-09+at+8.12.58+pm.png)


on iterm: 


    Asjads-MacBook-Pro:documents asjad$ ssh -i "mimic2.pem" ubuntu@ec2-34-229-64-213.compute-1.amazonaws.com

#### 4.  install screen: 

For accessing remote servers screen is a useful utility for acting as a container and manager for other terminal sessions. It can let you connect to multiple remote terminal sessions. 

we use apt-get to install: 

    sudo apt-get update
    sudo apt-get install screen

start a new screen session:


    screen

versify if its running properly 

    Ctrl-a v


#### 5. Install Git and Clone the repo: 


    clone: https://github.com/MLforHealth/MIMIC_Extract.git

we place the data in the following directory for now: 


    cd ~/MIMIC_Extract/data 


#### 6. Fetching the data: 

download the dataset(enter the Physionet password on prompt): 


    wget -r -N -c -np --user asjad99 --ask-password https://physionet.org/files/mimiciii/1.4/

This will take a few mins to complete (~6.2 GB)

(optional) The files will compressed and we can uncompress them using: 


    gunzip *.gz



#### 7. Build SQL DB: 

as a first step install postgres: 


    sudo apt-get update
    
    sudo -u postgres createuser --interactive
    Enter name of role to add: (same as OS user)
    
    #create a DB
    createdb mimic
    
    #connect:
    psql -U ubuntu -d mimic
    
     \c mimic;
    CREATE SCHEMA mimiciii;
    
    set search_path to mimiciii;
    SET
    

~/MIMIC_Extract/data/physionet.org/files/mimiciii/1.4



#### 8. Create tables: 

Then clone this repo:


    https://github.com/MIT-LCP/mimic-code/

see  script /buildmimic/postgres/postgres_create_tables.sql

 in the MIMIC code repository to create the mimiciii schema and then build a set of empty tables.


    psql 'dbname=mimic user=mimicuser options=--search_path=mimiciii' -f postgres_create_tables.sql

output: 

List all table by doing 


    \dt


#### 9. Populate the tables: 

we use the following script to populate tables: buildmimic/postgres/postgres_load_data_gz.sql

we run it 


    psql 'dbname=mimic user=mimicuser options=--search_path=mimiciii' -f postgres_load_data.sql -v mimic_data_dir='<path_to_data>'

output: 
COPY 58976
COPY 34499
COPY 7567
COPY 0
... etc

to monitor performance use:


    htop



we can check their sizes to get a rough idea that the copy operation was successful: 


    SELECT *, pg_size_pretty(total_bytes) AS total 
       , pg_size_pretty(index_bytes) AS index
        , pg_size_pretty(toast_bytes) AS toast
        , pg_size_pretty(table_bytes) AS table
      FROM (
      SELECT *, total_bytes-index_bytes-coalesce(toast_bytes,0) AS table_bytes FROM (
          SELECT c.oid,nspname AS table_schema, relname AS table_name
                  , c.reltuples AS row_estimate
                  , pg_total_relation_size(c.oid) AS total_bytes
                  , pg_indexes_size(c.oid) AS index_bytes
                  , pg_total_relation_size(reltoastrelid) AS toast_bytes
              FROM pg_class c
              LEFT JOIN pg_namespace n ON n.oid = c.relnamespace
              WHERE relkind = 'r'
      ) a
    ) a;
Create Materialised views: 

 
 first run this file: 
 https://github.com/MIT-LCP/mimic-code/blob/master/concepts/postgres-functions.sql
 
 then run 
concepts/postgres_make_concepts.sh 

----------------------------------------------------

