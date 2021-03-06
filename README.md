## Summary 

**Note: Please consider just branch V2 of this project as it is more recently updated**
Find it here (its the main branch) </br>
https://github.com/DanielePalaia/gpss-rabbit-greenplum-connector

</br>

This software is intended to be a simple (non production ready) connector rabbitmq-greenplum, similar to the default gpsscli which is supporting kafka.</br>

It is based on gpss (greenplum streaming server) so will work just with greenplum 5.16 or above.
https://gpdb.docs.pivotal.io/5160/greenplum-stream/overview.html </br>

GPSS is basically a grpc server, so this is pratically a grpc client written in GO.</br>


The connector is able to use the full parallel loading capability offered by Greenplum through the gpfdist protocol </br>

The connector will attach to a rabbitmq queue specified at configuration time will batch a certain amount of elements specified in memory and will then ask the gpss server to push them on a greenplum table.

In the other branch of this repo, a persistent option was added in order to restore the server in case of crash without loosing items, also some log improvements was implemented.

These are the steps to run the software:

## Prerequisites:

1. Activate the gpss extension on the greenplum database you want to use (for example test)<br/><br/>
   **test=# CREATE EXTENSION gpss;**<br/><br/>
   
2. create a table inside this database with a json field on it (for example mytest3)<br/><br/>
   **test=# create table mytest3(data json);**<br/><br/>
   
   Update: Originally the grpc client was working just with json, now it is able to automatically ask to the server for table definition and manage different kin of datatypes; so you can specify the table definition you want<br/><br/>
   
   **test=# create table companies(id varchar 200, city varchar 200, foundation timestamp, description text, data json);<br/><br/>**
   
   ![Screenshot](definition.png)
   
   
3. Run a gpss server with the right configuration (ex):<br/><br/>
  **gpss ./gpsscfg1.json --log-dir ./gpsslogs** <br/><br/>
  where gpsscfg1.json is <br/><br/>
  {<br/>
    "ListenAddress": {<br/>
        "Host": "",<br/>
        "Port": 50007,<br/>
        "SSL": false<br/>
    },<br/>
    "Gpfdist": {<br/>
        "Host": "",<br/>
        "Port": 8086<br/>
    }<br/>
}<br/><br/>

4. download, install and run a rabbitmq broker<br/><br/>
 **./rabbitmq-server**

5. Create a rabbitmq durable queue with the rabbitmq UI interface you want the connector to connect (es gpss):<br/>
  ![Screenshot](queue.png)<br/> </br>
  
## Compiling and Installing the application </br> 

The application is written in GO. Binary for MacosX and Linux are already provided inside the /bin folder. <br/>
If you need to compile and install it you need to download a GO compiler (ex for Linux - ubuntu) </br>

1. sudo apt-get install golang-go <br>
2. export GOPATH=/home/user/GO <br>
3. create a directory src inside GO and go there </br>
4. git clone https://github.com/DanielePalaia/gpss-rabbit-greenplum-connector and enter the project</br>
5. go get github.com/golang/protobuf/proto </br>
   go get github.com/streadway/amqp </br>
   go get google.golang.org/grpc </br>
   cp -fR ./gpss /home/user/GO/src/gpssclient </br>
6. go install gss-rabbit-greenplum-connector and you will find your binary in GOPATH/bin </br> </br>
  
## Running the application:<br/>

1. The application is written in GO. If you are using MacOs then you can directly use the binary version inside /bin of this project called: gpss-rabbit-greenplum-connect otherwise you must compile it with the GO compiler<br/>

2. Use the file properties.ini (that should be place in the same directory of the binary in order to instruct the program with this properties<br/>

    **GpssAddress=10.91.51.23:50007**<br/>
    **GreenplumAddress=10.91.51.23**<br/>
    **GreenplumPort=5533**<br/>
    **GreenplumUser=gpadmin**<br/>
    **GreenplumPasswd=**<br/> 
    **Database=test**<br/>
    **SchemaName=public**<br/>
    **TableName=mytest3**<br/>
    **rabbit=amqp://guest:guest@localhost:5672/**<br/>
    **queue=gpss**<br/>
    **batch=10000** <br/>

queue is the rabbitmq queue name while batch is the amount of batching that the rabbit-greenplum connector must take before pushing the data into greenplum.<br/>

3. Run the connector:<br/>
**./gpss-rabbit-greenplum-connect**<br/> 
**Danieles-MBP:bin dpalaia$ ./gpss-rabbit-greenplum-connector **<br/>
**connecting to grpc server**<br/>
**connected**<br/>
**2019/02/26 17:01:30  [*] Waiting for messages. To exit press CTRL+C**<br/>

4. Populate the queue payload with the UI interface. Every line is a field (write NULL for Nullable values) so for example (companies table):<br/><br/>
![Screenshot](queue3.png)

5. Once you publish more messages than the batch value you should then see the table populated and you can restart publishing.<br/>

6. This is the result you should see:<br/>

**test=# select * from companies limit 3;**<br/><br/>
**id  | city |     foundation      | description |                           data  <br/>**                         
**------+------+---------------------+-------------+----------------------------------------------------------<br/>**
**777  | Rome | 2017-08-19 12:17:55 | my description | { "cust_id": 1313131, "month": 12, "expenses": 1313.13 }<br/>**
**777  | Rome | 2017-08-19 12:17:55 | my description | { "cust_id": 1313131, "month": 12, "expenses": 1313.13 }<br/>**
**777  | Rome | 2017-08-19 12:17:55 | my description | { "cust_id": 1313131, "month": 12, "expenses": 1313.13 }<br/>**

7. In order to make tests easy I also developed a simple rabbitmq producer inside rabbit-client, you can find a binary for macos always inside bin.
If you run<br/>
**./rabbit-clientEX2**<br/>
he will take the same configuration that is inside properties.ini and will start to fire messages inside the same queue.
In this way some raliable tests can be done.
