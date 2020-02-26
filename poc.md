                        ******DEBEZIUM CDC with Kafka for DB AUDITING******



Debezium CDC with Kafka for DB
Auditing



-- Debezium is built on top of Apache Kafka and provides Kafka Connect compatible connectors that monitor
specific database management systems.



-- Debezium records the history of data changes in Kafka logs
for operations on database.



--
Running Debezium involves three major services: ZooKeeper, Kafka, and
Debezium’s connector service.



Perquisite:- 



1.      AWS EC2 instance created and running



2.      Docker installed on EC2



                                



Steps:



1.      docker run –it --name postgres -p 5432:5432
debezium/postgres



--this will install postgres 



 



2.     
docker
run -it --name zookeeper -p 2181:2181 -p 2888:2888 -p 3888:3888
debezium/zookeeper



--this will install zookeeper



 



3.     
docker
run -it --name kafka -p 9092:9092 — link zookeeper:zookeeper debezium/kafka



--this will install Kafka



 



4.     
docker
run -it --name connect -p 8083:8083 -e GROUP_ID=1 -e
CONFIG_STORAGE_TOPIC=my-connect-configs -e
OFFSET_STORAGE_TOPIC=my-connect-offsets -e ADVERTISED_HOST_NAME=$(echo
$DOCKER_HOST | cut -f3 -d’/’ | cut -f1 -d’:’) --link zookeeper:zookeeper --link
postgres:postgres --link kafka:kafka debezium/connect



--this will install debezium
connector



 



**It can be seen in
Screenshot that all the docker containers created using above commands are up
and running.





5. Create databse in
postgresql container, using following commands



>docker exec  –it <container id>  bash



> psql -h localhost -p 5000 -U postgres



*Then
create db and table

CREATE DATABASE inventory;

CREATE TABLE
demo(id SERIAL PRIMARY KEY, name VARCHAR);





5.      Create connector using
Kafka Connect



--Our setup is ready now we need to register Kafka
connect to database



curl -X POST -H "Accept:application/json"
-H "Content-Type:application/json" localhost:8083/connectors/ -d 



'{



 "name":
"inventory-connector",



 "config": {



 "connector.class":
"io.debezium.connector.postgresql.PostgresConnector",



 "tasks.max": "1",



 "database.hostname": "172.17.0.1",



 "database.port": "5432",



 "database.user":
"postgres",



 "database.password":
"postgres",



 "database.dbname":
"inventory",



 "database.server.name":
"dbserver1",



 "database.whitelist":
"inventory",



 "database.history.kafka.bootstrap.servers":
"kafka:9092",



 "database.history.kafka.topic":
"schema-changes.inventory"



 



 }



}'



Note: - If getting the error after giving
this command: Kindly check hostaname and port



{"error_code":500,"message":"Could
not create PG connection"}



Run > “ifconfig” to know the hostname
and other details, and change hostname as specified



Note:-Check
out the logs for debezium/connect container>docker logs “Container id” 



**Connector is registered with name
inventory-connector





6.     
curl -X GET -H "Accept:application/json" localhost:8083/connectors/inventory-connector



-- Verify the Connector is
created





>curl  -H "Accept:application/json"
localhost:8083/connectors/inventory



-for confirmation





7.      docker run -it --name watcher --rm --link zookeeper:zookeeper --link kafka:kafka debezium/kafka:0.10 watch-topic -a -k dbserver1.public.demo 

--Start a Kafka Console consumer to watch change

 

 

 

 

**inserting into table 

 

 

 

**All the containers are running including watcher

 

 

 

 

>docker logs <watcher container id>

--watcher logs to see the audit logging for dml operations on db or table

 

 

 

**To fetch json log file of Kafka watcher from docker container

root->cd var/lib/docker/containers

>cd containerid/

>cat containerid-json.logs

--for fetching the json log file from docker container

 

                   

Requirement as on Verizon Hackathon page on oneconfluence.



Capture all the
key information – 



 



·        
What is the field getting changed



ü
Done-(can be seen in below screenshot)



 



·        
What is the old value



ü
Done (can be seen in below screenshot as “before”)



 



Note:-“before” as null for
update and delete because table is created with replica        identity as default, so alter table with
“replica identity as full”



>alter table table_name
replica identity full;



 



·        
What is the new value



ü
Done(can be seen in below screenshot as “after”)



·        
Timestamp to show it is changed 



ü
Done(can be seen in below screenshot as “ts_ms” mean timestamp in
millisecond)



 



·        
What type of DML
operation it is



ü
Done(can be seen in below screenshot
as “op” mean operation which is   ‘c’
means create, ‘u’ means ‘update’, etc.)



 



·        
Who performed the
operation,etc



v please do the validation





**Using AWS CloudWatch

Step1.1. GO to AWS CloudWatch>Open Log Groups>Click on Actions and Create log group

1.2. Give some name

 

 

 

 

 

** Log group is created in cloudwatch

 

**Now there is no log, since we have not integrated out watcher with AWS CloudWatch

 

 

 

 

 

Pre-requisite to integrate cloudwatch with kafka watcher:

>mkdir -p /etc/systemd/system/docker.service.d/

>touch /etc/systemd/system/docker.service.d/aws-credentials.conf

>vim /etc/systemd/system/docker.service.d/aws-credentials.conf

[Service]

Environment="AWS_ACCESS_KEY_ID”

Environment="AWS_SECRET_ACCESS_KEY”

>sudo systemctl daemon-reload

>sudo service docker restart

 

Note: Create docker containers again 

 

 

**Integrate CloudWatch with Kafka topic watcher to store the logs. Using below commands:

> docker run --log-driver="awslogs" --log-opt awslogs-region="us-east-2" --log-opt awslogs-group="POC_CDC" --log-opt awslogs-stream="cdcstream" -it --name watcher_poc --rm --link zookeeper:zookeeper --link kafka:kafka debezium/kafka:0.10 watch-topic -a -k dbserver1.public.poc

Note: use table which is exists in database

Note: aws-log-group-name should be same as in cloudwatch, also check the region

 

 

 

 

 

 

**watcher4 is running

 

**CloudWatch screen which contains updated watcher logs

 

 

 

 

 

**Convert AWS CloudWatch logs to Table in AWS DynamoDB**

 

Steps:

Steps1.1 Go to IAM>Policies>Create Policy

               

 

 

 

 

 

 

 

 

 

 

 

1.2. Add CloudWatch and DynamoDB policies and set as in screenshots     

 

 

 

 

 

 

 

1.3. Review Policy

 

 

 

 

 

 

1.4.Create Policy

 

**Policy is created

 

 

 

 

 

1.5. Go to Roles>Create Role

 

1.6.Select as in AWS Service>Select Lambda

 

 

 

 

 

1.7. Select Policy which we created and move next

 

1.8. Next with no changes

 

 

 

 

 

1.9. Give some name in Role Name>Create role

 

 

Step 2: Go to AWS Lambda

2.1.Create function

 

 

 

2.2. Select “Author from scratch”>Give some name

 

2.3. Select python in runtime and select existing role which we created in previous step and Click on Create function

 

 

 

 

2.4. Our lambda function is created

 

2.6 Add trigger (our trigger here will be Cloudwatch logs)

>select created log in Log group

--when POC_CDC will have any updated events then this lambda function will trigger

 

 

 

**Lambda function is created and configured for POC_CDC log group

2.7. Click on poc_cd_lambda and write lambda function and Save

** Lamdba function here get logs which is in JSON format from clodwatch logs and save it to DynamoDB as tables.

 

 

 

 

Step 3: Go to AWS DynamoDB

3.1 Create table

3.2 Give some table name>set primary key as number (since we are using txId as primary key)

 

*Table is created

 

Approach to convert json to table and save in database?

**Now whatever dml operation will be performed, cloudwatch log will be updated automatically, hence it will trigger lambda function every time and json will be converted and saved as tables in DynamoDB. 

All this operation will be performed automatically on performing DML operation.

Testing:

** DynamoDB cdc_table is empty before performing operation

 

 

1. Perform some DML operation in respective table, here we are using watcher for ‘poc’ table.

 

 

 

 

 

 

 

 

2. This operation will be logged into AWS CloudWatch in POC_CDC log group

 

 

 

 

 

 

 

3./aws/lambda/poc_cdc_lmd is created on first dml operation

4.Logs of “/aws/lambda/poc_lambda” log group which sends data to dynamoDB.

FINAL OUTPUT

5. Tables of AWS DynamoDB which shows logs as table (used required fields only) 

 

 

 

 

 

 

                               *****************END******************
