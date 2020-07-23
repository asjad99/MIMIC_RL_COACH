### ABOUT 

This experiment was done to demonstrate how batch RL for Decision Support for clinicians 

We cover teh following 


- Data Acquisition 
- Pre-Processing 
- Creating Trajectories (Episodes) and Feature Encoding
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

--------------------------------------------------------------------------------------------------------

### PreProcessing

For pre-processing we will rely on this code (a library built in an effort to promote reproducibility in research)

https://github.com/MLforHealth/MIMIC_Extract


library: 
paper: https://arxiv.org/pdf/1907.08322.pdf
This helps us with Data Extraction and Preprocessing. 



#### Step 1: modify 

https://github.com/MLforHealth/MIMIC_Extract/blob/455a2484c1fd2de3809ec2aa52897717379dc1b7/utils/setup_user_env.sh

and then do: 


    source ./setup_user_env.sh



#### Step 2: install and create conda enviornment


    # Go to home directory
    cd ~
    
    # You can change what anaconda version you want at 
    # https://repo.continuum.io/archive/
    wget https://repo.continuum.io/archive/Anaconda2-4.2.0-Linux-x86_64.sh
    bash Anaconda2-4.2.0-Linux-x86_64.sh -b -p ~/anaconda
    rm Anaconda2-4.2.0-Linux-x86_64.sh
    echo 'export PATH="~/anaconda/bin:$PATH"' >> ~/.bashrc 
    
    # Reload default profile
    source ~/.bashrc
    
    conda update conda


After installation we will create a new environment: 


    conda env create --force -f ../mimic_extract_env.yml
     source activate mimic_data_extraction

Now we have to make a few edit in the mimic_direct_extract.py file: 

Firstly line 400 we change ventilationdurations to ventilation_durations

Similarly Edit the following line  439 from: 


    ``` table_names = ['vasopressordurations', 'adenosinedurations', 'dobutaminedurations', 'dopaminedurations', 'epinephrinedurations', 'isupreldurations', 
        #                'milrinonedurations', 'norepinephrinedurations', 'phenylephrinedurations', 'vasopressindurations']```

to


     ``` table_names = ['vasopressor_durations', 'adenosine_durations', 'dobutamine_durations', 'dopamine_durations', 'epinephrine_durations', 'isuprel_durations', 
                        'milrinone_durations', 'norepinephrine_durations', 'phenylephrine_durations', 'vasopress_indurations']```

  

After that run (preferably under screen): 
``` make build_curated_from_psql```


make sure enough space on hardrive: 

   ``` lsblk  sudo growpart /dev/nvme0n1 1 ```


#### Final Output: 


![](https://paper-attachments.dropbox.com/s_9C453E0E6A99C23A853A12A22A94087A14951068B3D837D1A2BDE0BFBFDEAD7A_1594286931904_Screen+Shot+2020-07-09+at+7.28.34+pm.png)


From the MIMIC relational database, SQL query results are processed to generate four output tables. 



see https://arxiv.org/pdf/1907.08322.pdf for details

--------------------------------------------------------------------------------------------------------

### Creating Trajectories (Episodes) and Feature Encoding

See the Notebook MIMIC_RL 

Input: 

we provide the input with follwoing files(obtained from last step)

```
X = pd.read_hdf(DATAFILE,'vitals_labs')
Y = pd.read_hdf(DATAFILE,'interventions')
static = pd.read_hdf(DATAFILE,'patients')

```

output: 

```

action,all_action_probabilities,episode_id,episode_name,reward,transition_number,state_feature_0,state_feature_1,state_feature_2,state_feature_3,state_feature_4,state_feature_5,state_feature_6,state_feature_7,state_feature_8,state_feature_9,state_feature_10,state_feature_11,state_feature_12,state_feature_13,state_feature_14,state_feature_15,state_feature_16,state_feature_17,state_feature_18,state_feature_19,state_feature_20,state_feature_21,state_feature_22
1,"[1, 1, 1, 1]",3,mimic_RL,0,0,0.4766666666666667,0.2770745804541222,0.74,0.05333333333333334,0.30600214362272243,0,0,1,0,0,0,1,0,0,0,0,0,1,0,0,1,0,0
3,"[1, 1, 1, 1]",3,mimic_RL,0,1,0.5125,0.13713081275360495,0.74,0.051666666666666666,0.2604501607717042,0,0,1,0,0,0,1,0,0,0,0,0,1,0,0,1,0,0
3,"[1, 1, 1, 1]",3,mimic_RL,0,2,0.45916666666666667,0.1557665092532645,0.985,0.023333333333333334,0.2915326902465166,0,0,1,0,0,0,1,0,0,0,0,0,1,0,0,1,0,0

```

--------------------------------------------------------------------------------------------------------


### Modeling with Intel Coach



#### prerequisites for intel coach: 


```

# General
sudo -E apt-get install python3-pip cmake zlib1g-dev python3-tk python-opencv -y

# Boost libraries
sudo -E apt-get install libboost-all-dev -y

# Scipy requirements
sudo -E apt-get install libblas-dev liblapack-dev libatlas-base-dev gfortran -y

# PyGame
sudo -E apt-get install libsdl-dev libsdl-image1.2-dev libsdl-mixer1.2-dev libsdl-ttf2.0-dev
libsmpeg-dev libportmidi-dev libavformat-dev libswscale-dev -y

# Dashboard
sudo -E apt-get install dpkg-dev build-essential python3.5-dev libjpeg-dev  libtiff-dev libsdl1.2-dev libnotify-dev 
freeglut3 freeglut3-dev libsm-dev libgtk2.0-dev libgtk-3-dev libwebkitgtk-dev libgtk-3-dev libwebkitgtk-3.0-dev
libgstreamer-plugins-base1.0-dev -y

# Gym
sudo -E apt-get install libav-tools libsdl2-dev swig cmake -y

```


#### Virtualenv installation: 

```
sudo -E pip3 install virtualenv
virtualenv -p python3 coach_env
. coach_env/bin/activate

```

### Install from repo: 


```
git clone https://github.com/NervanaSystems/coach.git

cd coach
pip3 install -e .

```

### Modeling 

see Notebook: 

input: /coach/

output: ./tmp/

3 kinds of ouputs available to us: 


we can use tensorboard to visualize the training process  

checkpoints 
