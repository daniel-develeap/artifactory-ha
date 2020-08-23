# artifactory-ha
documentation of artifactory upgrading &amp; migrating from ec2 standalone instance version 6.13.1 with docker-compose topology (artifactory, postgresql, nginx)  with AWS EBS volume as storage backend to artifactory-ha version 7.5.5 helm deployment on k8s topology with s3 as storage backend and AWS Aurora DB Postgresql as database

Upgrading docker-compose ec2 tenant from version 6.13.1 to 7.5.7 (currently all services volumes are mounted on /data .e.g /data/nginx /data/artifactory):
* Make sure to download artifactory packages according to required version & subscription type: https://www.jfrog.com/confluence/display/JFROG/Upgrading+Artifactory


ssh centos@<instance-ip>

sudo chown centos:centos /data
sudo scp -r -i <instance's-private-key> ./artifactory-pro-7.5.7/ centos@<instance-ip>:/data/

ssh -i <instance's-private-key> centos@<instance-ip>

cd /data

sudo chown root:root /data


sudo cp artifactory-pro-7.5.7/.env .

sudo chown centos:centos .env


sudo echo -e "JF_SHARED_NODE_IP=$(hostname -i)" >> .env
sudo echo -e "JF_SHARED_NODE_ID=$(hostname -s)" >> .env
sudo echo -e "JF_SHARED_NODE_NAME=$(hostname -s)" >> .env

* edit 'docker-compose.yaml' and modify values: change ART_BASE_URL variable according to your DNS address, and environment variables for each service according to yours

#
version: '2'
services:
  nginx:
    image: docker.bintray.io/jfrog/nginx-artifactory-pro:${ARTIFACTORY_VERSION}
    container_name: nginx
    ports:
     - 80:80
     - 443:443
    depends_on:
     - artifactory
    links:
     - artifactory
    volumes:
     - /data/nginx:/var/opt/jfrog/nginx
    environment:
     - ART_BASE_URL=http://XXX:8081/artifactory
     - SSL=true
    restart: always
    ulimits:
      nproc: 65535
      nofile:
        soft: 32000
        hard: 40000
  postgres:
    image: ${DOCKER_REGISTRY}/postgres:9.5.2
    container_name: postgresql
    environment:
     - POSTGRES_DB=artifactory
     - POSTGRES_USER=artifactory
     - POSTGRES_PASSWORD=XXX
    ports:
      - 5432:5432
    volumes:
     - /data/postgresql:/var/lib/postgresql/data # postgresql:/var/lib/postgresql/data
     - /etc/localtime:/etc/localtime:ro
    restart: always
    logging:
      driver: json-file
      options:
        max-size: "50m"
        max-file: "10"
    ulimits:
      nproc: 65535
      nofile:
        soft: 32000
        hard: 40000
  artifactory:
    image: ${DOCKER_REGISTRY}/jfrog/artifactory-pro:${ARTIFACTORY_VERSION}
    container_name: artifactory
    volumes:
     - /data/artifactory:/var/opt/jfrog/artifactory # artifactory:/var/opt/jfrog/artifactory
     - /etc/localtime:/etc/localtime:ro
    restart: always
    depends_on:
    - postgres
    ulimits:
      nproc: 65535
      nofile:
        soft: 32000
        hard: 40000
    environment:
     - ENABLE_MIGRATION=y
     - JF_SHARED_DATABASE_TYPE=postgresql
     - JF_SHARED_DATABASE_USERNAME=artifactory
     - JF_SHARED_DATABASE_PASSWORD=XXX    
     - JF_SHARED_DATABASE_URL=jdbc:postgresql://postgresql:5432/artifactory
     - JF_SHARED_DATABASE_DRIVER=org.postgresql.Driver
     - JF_SHARED_NODE_IP=${JF_SHARED_NODE_IP}
     - JF_SHARED_NODE_ID=${JF_SHARED_NODE_ID}
     - JF_SHARED_NODE_NAME=${JF_SHARED_NODE_NAME}
     - JF_ROUTER_ENTRYPOINTS_EXTERNALPORT=${JF_ROUTER_ENTRYPOINTS_EXTERNALPORT}
     - EXTRA_JAVA_OPTIONS=-Xms64g -Xmx64g
    ports:
      - ${JF_ROUTER_ENTRYPOINTS_EXTERNALPORT}:${JF_ROUTER_ENTRYPOINTS_EXTERNALPORT} # for router communication
      - 8081:8081 # for artifactory communication
    logging:
      driver: json-file
      options:
        max-size: "50m"
        max-file: "10"
#


sudo docker stop artifactory postgresql

sudo docker rm -f artifactory postgresql

**DOWNTIME PHASE - 5-10 MIN**

sudo docker-compose -p rt up -d


#  filestore migration to S3 - *** INCLUDES ADDITIONAL DOWNTIME

sudo docker exec -it artifactory /bin/bash

mkdir -p /var/opt/jfrog/artifactory/data/artifactory/eventual/
ln -s /var/opt/jfrog/artifactory/data/artifactory/filestore /var/opt/jfrog/artifactory/data/artifactory/eventual/_add
ln -s /var/opt/jfrog/artifactory/data/artifactory/filestore/_pre /var/opt/jfrog/artifactory/data/artifactory/eventual/_pre

# configure binarystore.xml according to "s3-storage-v3" template

- vi /opt/jfrog/artifactory/var/etc/artifactory/binarystore.xml

<config version="2">
<chain template="s3-storage-v3"/>
<provider id="s3-storage-v3" type="s3-storage-v3">
<endpoint>http://s3.amazonaws.com</endpoint>
<roleName>XXX</roleName>
<bucketName>XXX</bucketName>
<path>artifactory/filestore</path>
<region>us-east-1</region>
<identity>XXX</identity>
<credential>XXX</credential>
<usePresigning>true</usePresigning>
<signatureExpirySeconds>600</signatureExpirySeconds>
</provider>
</config>

sudo docker restart artifactory

**DOWNTIME PHASE - 5 MIN**

sudo docker logs artifactory


# initiate export process:

login to UI:
disable garbage collector through 'administration - artifactory' → 'maintanance' → 'garbage collection' → 'cron expression'

12 0 /4 * 12 ?


'administration - artifactory' → 'import & export' → 'system' → 'export system' → 'export path on server' → 'browse' → select path '/var/opt/jfrog/artifactory/system_export_8_8_20'
select 'exclude content' checkbox
select  'output verbose log' checkbox 
start export

watch export logs: 

sudo docker exec -it artifactory /bin/bash

tail -f var/log/artifactory-import-export.log

watch exported data directory size: du  -sh var/system_export_8_8_20/


# artifactory-ha binarystore.xml
/opt/jfrog/artifactory/var/etc/artifactory/binarystore.xml



# artifactory-ha-test helm deployment - INCLUDING *.sentinelone.net certificate (see 'nginx' section in values.yaml under '/Users/danielh/Desktop/sentinelOne/Tasks/artifactory-ha/artifactory-ha')

create mysecret.yaml with the following content:

apiVersion: v1

data:

  master-key: NTBhMDJiYmJkYjYyMmExMjhlM2EyYjBmZGY3NjkxYWM4MWJkYzVlOTZlODMzMzRjYzdjMzVkNzc3YWQ2NDYxNw==

kind: Secret

metadata:

  annotations:

    kubectl.kubernetes.io/last-applied-configuration: |

      {"apiVersion":"v1","data":{"master-key":"NTBhMDJiYmJkYjYyMmExMjhlM2EyYjBmZGY3NjkxYWM4MWJkYzVlOTZlODMzMzRjYzdjMzVkNzc3YWQ2NDYxNw=="},"kind":"Secret","metadata":{"annotations":{},"creationTimestamp":"2020-07-23T13:37:47Z","name":"my-secret","namespace":"artifactory-ha-stage","resourceVersion":"2312290891"},"type":"Opaque"}

  creationTimestamp: "2020-07-30T08:12:31Z"

  name: my-secret

  namespace: artifactory-ha-stage

  resourceVersion: "2386393811"

  selfLink: /api/v1/namespaces/artifactory-ha-stage/secrets/my-secret

  uid: d4c99366-cdfe-4f87-b5cd-fca13ce147de

type: Opaque

# apply created secret yaml file: kubectl create -f mysecret.yaml

# prepare AWS RDS AuroraPostgreSQL DB (version 9.6.11, DB cluster ID: artifactory-ha-aurora-stage, username: postgres, password: dWnjQFaYP9PjsAbW, initial DB name: artifactoryhadb, subnet: us-east-1-private-aurora, security group: artifactory-ha-aurora-sg ): https://console.aws.amazon.com/rds/home?region=us-east-1#database:id=artifactory-ha-aurora-stage;is-cluster=true;tab=connectivity 

# prepare env variables and deploy:

export AWS_REGION='us-east-1'
export AWS_S3_BUCKET_NAME='s1-artifactory-backend'
export AWS_IAM_ROLE_ARN='arn:aws:iam::161638504285:role/artifactory-role'



helm3 upgrade --install artifactory-ha-stage --set artifactory.masterKeySecretName=my-secret --set artifactory.persistence.type=aws-s3-v3 --set artifactory.persistence.awsS3V3.region=${AWS_REGION} --set artifactory.persistence.awsS3V3.bucketName=${AWS_S3_BUCKET_NAME} --set artifactory.annotations.'iam\.amazonaws\.com/role'=${AWS_IAM_ROLE_ARN} --set artifactory.persistence.enabled=false --set artifactory.persistence.awsS3.path=artifactory/filestore --set postgresql.enabled=false --set database.type=postgresql --set database.driver=org.postgresql.Driver --set database.url='jdbc:postgresql://artifactory-ha-aurora-stage.cluster-c8f6rvycvy9w.us-east-1.rds.amazonaws.com:5432/artifactoryhadb' --set database.user=postgres --set database.password=dWnjQFaYP9PjsAbW --set artifactory.node.replicaCount=0 --namespace artifactory-ha-stage --set artifactory.node.replicaCount=0 --set artifactory.service.pool=all --set artifactory.primary.resources.requests.cpu="8" \
--set artifactory.primary.resources.limits.cpu="16" \
--set artifactory.primary.resources.requests.memory="20Gi" \
--set artifactory.primary.resources.limits.memory="24Gi" \
--set artifactory.primary.javaOpts.xms="20g" \
--set artifactory.primary.javaOpts.xmx="20g" \
--set artifactory.node.resources.requests.cpu="2" \
--set artifactory.node.resources.limits.cpu="4" \
--set artifactory.node.resources.requests.memory="4Gi" \
--set artifactory.node.resources.limits.memory="6Gi" \
--set artifactory.node.javaOpts.xms="32g" \
--set artifactory.node.javaOpts.xmx="32g" \
--set nginx.resources.requests.cpu="100m" \
--set nginx.resources.limits.cpu="250m" \
--set nginx.resources.requests.memory="250Mi" \
--set nginx.resources.limits.memory="500Mi" artifactory-ha

# login to artifactory-ha UI: 

kubectl get all --namespace artifactory-ha-stage

copy 'service/artifactory-ha-stage-nginx' external IP address and paste in the browser

login with default creds: 'admin', 'password' and change password.
apply enterprise license key
disable garbage collection

import process: 'administration - artifactory' → 'import & export' → 'system' → 'import system' → 'browse' → select path '/var/opt/jfrog/artifactory/system_export_8_8_20'
select  'output verbose log' checkbox 
start import
