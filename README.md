# Batch Reinforcement Learning for Decision Support in Clinical Settings

This project demonstrates how **Batch Reinforcement Learning (RL)** can be effectively used for **Decision Support in Clinical Settings**. It explores the acquisition, processing, modeling, and application of clinical data for RL-based decision-making.

For more details, you can read the accompanying [blog post](https://blog.asjadk.com/decision_support/).

## Table of Contents

- [About](#about)
- [Data Acquisition](#data-acquisition)
- [Pre-Processing](#pre-processing)
- [Creating Trajectories (Episodes) and Feature Encoding](#creating-trajectories-episodes-and-feature-encoding)
- [Modeling with Intel Coach](#modeling-with-intel-coach)
- [Installation and Setup](#installation-and-setup)
- [Experimental Evaluation](#experimental-evaluation)
- [Outputs](#outputs)

## About

The aim of this experiment is to demonstrate how batch reinforcement learning can be used to support clinical decision-making. The pipeline includes:

- Data Acquisition
- Data Pre-Processing
- Creating Trajectories (Episodes) and Feature Encoding
- Modeling with Reinforcement Learning

## Data Acquisition

### 1. Request Access to Acquire the Data

The data used in this project comes from the **MIMIC-III** database, which contains health-related records for critical care patients. To acquire this data, follow these steps:

1. Create an account on [PhysioNet](https://physionet.org/) and request access to the dataset: [MIMIC-III Clinical Database v1.4](https://physionet.org/content/mimiciii/1.4/).
2. Review the schema of the database here: [MIMIC Schema](https://mimic.physionet.org/mimicdata/schema/).

### 2. Launch an AWS Instance for Analysis

If you don't have a data engineering team available to set up data pipelines, it's useful to have a quick way to access powerful computing resources.

Follow this guide on how to set up an AWS EC2 instance for data analysis: [Zero to AWS EC2 for Data Science](https://medium.com/@junseopark/from-zero-to-aws-ec2-for-data-science-62e7a22d4579).

### 3. Connect to the EC2 Instance

To connect to the AWS instance, use the command below (using Transmit and iTerm simultaneously is recommended for easy file transfers).

```sh
ssh -i "mimic2.pem" ubuntu@<your-ec2-instance-address>
```

### 4. Install Screen for Remote Sessions

**Screen** is a useful utility for managing remote terminal sessions, allowing you to keep multiple terminal sessions open.

Install Screen using the following commands:

```sh
sudo apt-get update
sudo apt-get install screen
screen
```

Verify that the screen session is running properly by pressing `Ctrl-a v`.

### 5. Install Git and Clone Required Repositories

To download the necessary code:

```sh
git clone https://github.com/MLforHealth/MIMIC_Extract.git
cd ~/MIMIC_Extract/data
```

### 6. Fetch the Data

Use **wget** to download the MIMIC-III dataset (enter your PhysioNet credentials when prompted):

```sh
wget -r -N -c -np --user <your-username> --ask-password https://physionet.org/files/mimiciii/1.4/
```

(Optional) Uncompress the files:

```sh
gunzip *.gz
```

### 7. Build SQL Database

Install **PostgreSQL** and create a database for storing the MIMIC-III data:

```sh
sudo apt-get update
sudo apt-get install postgresql

sudo -u postgres createuser --interactive
createdb mimic
```

Connect to the PostgreSQL database and set up the schema:

```sh
psql -U ubuntu -d mimic
\c mimic;
CREATE SCHEMA mimiciii;
set search_path to mimiciii;
```

### 8. Create Tables and Populate Data

Clone the **MIMIC Code** repository:

```sh
git clone https://github.com/MIT-LCP/mimic-code/
```

Create tables using the SQL script provided:

```sh
psql 'dbname=mimic user=mimicuser options=--search_path=mimiciii' -f postgres_create_tables.sql
```

To populate the tables:

```sh
psql 'dbname=mimic user=mimicuser options=--search_path=mimiciii' -f postgres_load_data.sql -v mimic_data_dir='<path_to_data>'
```

Check the sizes of the tables to verify that the data has been correctly populated.

### 9. Create Materialized Views

Run the following script to create materialized views:

1. [Postgres Functions Script](https://github.com/MIT-LCP/mimic-code/blob/master/concepts/postgres-functions.sql)
2. Then run: `concepts/postgres_make_concepts.sh`

## Pre-Processing

The preprocessing step involves data extraction and feature engineering to create suitable input for modeling.

### Step 1: Modify User Environment Setup

Edit the user environment setup script as needed:

```sh
https://github.com/MLforHealth/MIMIC_Extract/blob/455a2484c1fd2de3809ec2aa52897717379dc1b7/utils/setup_user_env.sh
```

Source the script:

```sh
source ./setup_user_env.sh
```

### Step 2: Install Anaconda and Set Up Environment

Install Anaconda and create a new environment for MIMIC data extraction:

```sh
cd ~
wget https://repo.continuum.io/archive/Anaconda2-4.2.0-Linux-x86_64.sh
bash Anaconda2-4.2.0-Linux-x86_64.sh -b -p ~/anaconda
rm Anaconda2-4.2.0-Linux-x86_64.sh
echo 'export PATH="~/anaconda/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

conda update conda
conda env create --force -f ../mimic_extract_env.yml
source activate mimic_data_extraction
```

Edit the `mimic_direct_extract.py` file for specific column names, and then build the curated dataset:

```sh
make build_curated_from_psql
```

Ensure enough space is available:

```sh
lsblk
sudo growpart /dev/nvme0n1 1
```

## Creating Trajectories (Episodes) and Feature Encoding

To create episodes and encode features, use the **MIMIC_RL** notebook.

### Input

The input to the model is obtained from the last step:

```python
X = pd.read_hdf(DATAFILE, 'vitals_labs')
Y = pd.read_hdf(DATAFILE, 'interventions')
static = pd.read_hdf(DATAFILE, 'patients')
```

### Output

The generated output is a set of RL trajectories:

```
action,all_action_probabilities,episode_id,episode_name,reward,...
```

## Modeling with Intel Coach

The modeling phase involves using **Intel Coach** for RL experiments.

### Prerequisites for Intel Coach

```sh
# Install necessary packages
sudo -E apt-get install python3-pip cmake zlib1g-dev python3-tk python-opencv -y
sudo -E apt-get install libboost-all-dev -y
sudo -E apt-get install libblas-dev liblapack-dev libatlas-base-dev gfortran -y
sudo -E apt-get install libsdl-dev libsdl-image1.2-dev ...
```

### Virtualenv Installation

```sh
sudo -E pip3 install virtualenv
virtualenv -p python3 coach_env
. coach_env/bin/activate
```

### Install from Repository

```sh
git clone https://github.com/NervanaSystems/coach.git
cd coach
pip3 install -e .
```

## Experimental Evaluation

In our experiments, we focus on several popular RL algorithms, including:

- **Deep Q Learning (DQN)**
- **Double Deep Q Learning (DDQN)**
- **DDQN combined with Bootstrapped Neural Coach (BNC)**
- **Mixed Monte Carlo (MMC)**
- **Persistent Advantage Learning (PAL)**

Refer to the [MIMIC RL Notebook](https://github.com/asjad99/MIMIC_RL_COACH/blob/master/MIMIC_RL.ipynb) for further details on the experimental setup.

### Machine Specs
Provide specifications of the machine used for running the experiments.

## Outputs

We rely on various **Off-Policy Evaluation (OPE)** metrics to evaluate the performance of the trained RL models without deploying them.

- **Training Logs**: Use **TensorBoard** to visualize the training process.
- **Checkpoints**: Save model checkpoints for evaluation and comparison.

---
Feel free to explore the repository for additional details or reach out via issues for questions and discussions!

