# fitness-data-project
<b>Demo on analyzing and aggregating fitness data on google cloud platform.</b>

This project contains the code necessary to setup either a Google Cloud Compute Engine VM with a data simulation script in order to demonstrate a data pipeline involving the following google cloud platform services.

  1. Google Cloud Iot Core
  2. Google Cloud Compute Engine
  3. Google Cloud Pub/Sub
  4. Google Cloud Dataflow
  5. Google Cloud BigQuery
  6. Google Cloud Storage

The instructions below use the Google Cloud SDK tools to access Google Cloud Platform products and services from the command line.

## Setup

### Login to Google Cloud.  Create new project(this project was named 'fitness-data-project').

### Open Google Cloud Shell

### Enable the necessary APIs

        gcloud services enable compute.googleapis.com
        gcloud services enable dataflow.googleapis.com
        gcloud services enable pubsub.googleapis.com
        gcloud services enable cloudiot.googleapis.com

### Create a BigQuery dataset and table:

        bq --location=US mk -d fitnessdataset
        bq mk -t --description "Member fitness data" fitnessdataset.membertable date:DATE,steps:INT64,caloriesburned:INT64,caloriesconsumed:INT64,minutesofsleep:INT64
        
   In the case the table needs to be deleted (i.e. in order to be recreated)...
   
        bq rm -t -f fitnessdataset.membertable

### Create a PubSub topic(fitnesstopic):

        gcloud beta pubsub topics create projects/fitness-data-project/topics/fitnesstopic

### Create a Cloud Storage Bucket(fitnessb):

        gsutil mb -p fitness-data-project -c multi_regional -l US gs://fitnessb/


### Create a Dataflow process:

        Creating Dataflow templates from the command line is not yet supported. 
        1.GO TO THE CLOUD DATAFLOW WEB UI
        2.Create Job From Template
        3.Enter a Job name for your Cloud Dataflow Job: pubsubbigquery
        4.Under Cloud Dataflow template, select: Cloud Pub/Sub to BigQuery
        5.Under Cloud Pub/Sub input topic, enter: projects/fitness-data-project/topics/fitnesstopic
        6.Under BigQuery output table, enter: fitness-data-project:fitnessdataset.membertable
        7.Under Temporary Location, enter: gs://fitnessb/tmp
        8.Click the Run job button.

### Create a registry:

        gcloud iot registries create mobile-devices \
            --project=fitness-data-project \
            --region=us-central1 \
            --event-notification-config=topic=projects/fitness-data-project/topics/fitnesstopic

### Create a VM

        gcloud compute instances create mobile-device-simulator

### Connect to the VM. Install the necessary software and create a security certificate. Note the full path of the directory that the security certificate is stored in (the results of the "pwd" command). Then exit the connection.

         gcloud compute ssh mobile-device-simulator
         sudo apt-get update 
         sudo apt-get install python-pip openssl git google-cloud-sdk -y
         sudo pip install cryptography pyjwt paho-mqtt tendo
         git clone https://github.com/carlos-i-martinez/fitness-data-project
         cd fitness-data-project
         openssl ecparam -genkey -name prime256v1 -noout -out ~/.ssh/ec_private.pem
         openssl ec -in ~/.ssh/ec_private.pem -pubout -out ~/.ssh/ec_public.pem
         wget -O ~/.ssh/roots.pem https://pki.goog/roots.pem
         cd ../.ssh
         pwd
         exit

### Use SCP to copy the public key that was just generated. The path the SSH keys was the result of the "pwd" command in the previous step.

        gcloud compute scp mobile-device-simulator:/[PATH TO SSH KEYS]/ec_public.pem .

### Register the VM as an IoT device:

        gcloud beta iot devices create myVM \
            --project=fitness-data-project \
            --region=us-central1 \
            --registry=mobile-devices \
            --public-key path=ec_public.pem,type=es256

### Start the Data Simulator
### Connect to the VM. Send the mock data(sample.csv)using the mobiledevicesimulator.py script. This script changes the csv file to a json file(file1.json) and publishes JSON-formatted messages to the device's MQTT topic one by one:

        gcloud compute ssh mobile-device-simulator
        cd fitness-data-project
        python mobiledevicesimulator.py --registry_id=mobile-devices --project_id=fitness-data-project --device_id=myVM --private_key_file=../.ssh/ec_private.pem
        exit
        
    Exit from the Cloud Shell
    
        exit

### See the Data Flow
### Go to BigQuery, query the data.
```
SELECT * FROM fitness-data-project:fitnessdataset.membertable ORDER BY date DESC'
```
### Visualize the data. Visualization reporting mechanisms TBD.








### After you are done, clean everything up

        gcloud dataflow jobs list
        gcloud dataflow jobs cancel [JOB_ID]
        bq rm -r fitness-data-project:fitnessdataset
        gcloud beta pubsub topics delete fitnesstopic
        gcloud beta iot devices list --registry=mobile-devices --region=us-central1
        gcloud beta iot devices delete myVM
        gcloud beta iot registries delete mobile-devices --region=us-central1
        gcloud compute instances delete mobile-device-simulator
