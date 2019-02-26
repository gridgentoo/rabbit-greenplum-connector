This software is intended to be a simple (non production ready) connector rabbitmq-greenplum, similar to the default gpsscli which is supporting kafka.

It is based on gpss (greenplum streaming server) so will work just with greenplum 5.16 or above.
https://gpdb.docs.pivotal.io/5160/greenplum-stream/overview.html

The connector will attach to a rabbitmq queue specified at configuration time will batch a certain amount of elements specified and will ask the gpss server to push them on a greenplum table.

For the moment the connector is supporting just json format (elements must be strings of json in rabbitmq and the resulting greenplum table needs to be a table with a json field).

These are the steps to run the software:

Prerequisites:

1) Activate the gpss extension on the greenplum database you want to use (for example test)<br/>
   **test=# CREATE EXTENSION gpss;**
   
2) create a table inside this database with a json field on it (for example mytest3)
   **test=# create table mytest3(data json);**
   
3) Run a gpss server with the right configuration (ex):<br/>
  gpss ./gpsscfg1.json --log-dir ./gpsslogs <br/>
  where gpsscfg1.json is <br/><br/>
  {
    "ListenAddress": {
        "Host": "",
        "Port": 50007,
        "SSL": false
    },
    "Gpfdist": {
        "Host": "",
        "Port": 8086
    }
}<br/>

4) download, install and run a rabbitmq broker<br/>
 ./rabbitmq-server

5) Create a rabbitmq transient queue with the rabbitmq UI interface you want the connector to connect (es gpss):<br/>
  ![Screenshot](queue.png)
  
Running the application:

1) The application is written in GO. If you are using MacOs then you can directly use the binary version inside /bin of this project called: gpss-rabbit-greenplum-connect otherwise you must compile it with the GO compiler

2) Use the file properties.ini (that should be place in the same directory of the binary in order to instruct the program with this properties<br/>

GpssAddress=10.91.51.23:50007
GreenplumAddress=10.91.51.23
GreenplumPort=5533
GreenplumUser=gpadmin
GreenplumPasswd=**** 
Database=test
SchemaName=public
TableName=mytest3
rabbit=amqp://guest:guest@localhost:5672/
queue=gpss
batch=50000  

queue is the rabbitmq queue name while batch is the amount of batching that the rabbit-greenplum connector must take before pushing the data into greenplum.<br/>

3) Run the connector:<br/>
./gpss-rabbit-greenplum-connect 
Danieles-MBP:bin dpalaia$ ./gpss-rabbit-greenplum-connector 
connecting to grpc server
connected
2019/02/26 17:01:30  [*] Waiting for messages. To exit press CTRL+C<br/>

4) Populate the queue with the UI interface (Publish command)<br/>
![Screenshot](queue2.png)

5) Once you publish more messages than the batch value you should then see the table populated and you can restart publishing.<br/>

6) In order to make tests easy I also developed a simple consumer inside rabbit-client, you can find a binary for macos always inside bin.
If you run<br/>
./rabbit-client
he will take the same configuration that is inside properties.ini and will start to fire messages inside the same queue.
