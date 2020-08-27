# event-pipeline
![Image of Event Pipeline](https://miro.medium.com/max/700/1*eCRP1WjEY5tahxJK_e6VMg.png)

1) Create Kinesis
  > - Name= ‘event-pipe’
  > - Only 1 shard
  > - Create kinesis stream
2) Create API Gateway
  > - Name “Event-pipe”
  > - Create API
3) Create a role that allows API Gateway to write into Kinesis
  > - Create role -> api gateway -> name “api-gateway-pipeline”
  > - Attach another policy “Kinesis full access” (save ARN)
4) Add Resources and Methods in API Gateway
  > - Create URL path for REST API
      > - Name “Streams”
      > - Create resources
      > - Method
          > - Get Method
          > - Integration Type: AWS Service
          > - Region: us-east-1
          > - Service:  Kinesis
          > - HTTP method:  POST
          > - Action:  ListStreams
          > - Execution role:  ARN of the role we created.
          > - Save
      > - Checkout out Integration Request (tells Kinesis what kind of data we are dealing with) = in our case we want JSON data
          > - HTTP Headers
          > - Name “Content-Type” 
          > - Mapped from: ‘application/x-amz-json-1.1’  
          > - Mapping Templates - select (recommended)
          > - Content-type: application/json
          > - { }
          > - Save
      > - Go back to method execution and Test
          > - Test Get method - should list all available Kinesis streams
5) Create 2nd resource
    > - Resource Name = KinesisStream
    > - Resource Path = {stream-name}
    > - Create
    > - Method
        > - Get Method
        > - Integration Type: AWS Service
        > - Region: us-east-1
        > - Service:  Kinesis
        > - HTTP method:  POST
        > - Action:  DescribeStream
        > - Execution role:  ARN of the role we created.
        > - Save
      > - Checkout out Integration Request
        > - HTTP Headers
        > - Name “Content-Type” 
        > - Mapped from: ‘application/x-amz-json-1.1’  
        > - Mapping Templates - select (recommended)
        > - Content-type: application/json
        > - {  "StreamName": "$input.params('stream-name')" }
        > - Save
      > - Go back to method execution and Test
        > - Test Get method - this time add the name of the STREAM
6) Create another resource (This path will give Kinesis which eventually will be transferred to s3 bucket)
    > - Resource name “record”
    > - Enable API Gateway CORS
    > - Create Resource
    > - Method (PUT)
        > - Put Method
        > - Integration Type: AWS Service
        > - Region: us-east-1
        > - Service:  Kinesis
        > - HTTP method:  POST
        > - Action:  PutRecord
        > - Execution role:  ARN of the role we created.
        > - Save
      > - Checkout out Integration Request  
        > - HTTP Headers
        > - Name “Content-Type” 
        > - Mapped from: ‘application/x-amz-json-1.1’  
        > - Mapping Templates - select (recommended)
        > - Content-type: application/json
        > - { "StreamName": "$input.params('stream-name')",
             "Data": "$util.base64Encode($input.json('$.Data'))Cg==",
            "PartitionKey": "$input.path('$.PartitionKey')"
            }
        > - Save
          <ol>
            <li>Mapping template will have the actual Data that we want to pass through to an s3 bucket</li>
            <li>2nd will be a Stream Name - which we’ll get from our URL path</li>
            <li>PartitionKey - PartitionKey is an integer that’s hashed in order to arrive at a shard number
                <li>Example: if we have multiple shards`</li>
             </li>
          <ol>
            
      > - Go back to method execution and Test
          > - Add { “Data”: “testing”, “PartitionKey”: 1}
7) Since Kinesis Streams can’t write directly to s3, we need to create delivery stream in Kinesis Firehose  
    > - Service Kinesis
    > - Create Delivery Stream
    > - Delivery Stream Name “event-pipe-delivery”
    > - Select Kinesis stream & the stream we created -> next
    > - Leave Lambda & converting disable (for this project)
        > - We’ll use lambda next time to see what we can do to the data that reached the s3 bucket
    > - Destination s3 bucket
    > - Name “event-type-data-XXXX”
        > - Prefix will be by year,month, date, hour  -> next
    > - Buffer conditions
        > - Change buffer to 60 seconds for our small project
    > - Compression  - select GZIP
    > - Create a new ROLE
8) Next & Create delivery stream
9) Finally deploy api for testing
    > - Api gateway
    > - Select “event-pipe”
    > - Action -> Deploy API
        > - Test, test, test,test  -> deploy

We get the url

https://wle4ucytwl.execute-api.us-east-1.amazonaws.com/test
Test our url

![First Glance](https://github.com/sammycoderu/event-pipeline/blob/master/pipeline-pics/firsttestresult.png)



https://wle4ucytwl.execute-api.us-east-1.amazonaws.com/test/streams
List the streams
Pick one of the streams

https://wle4ucytwl.execute-api.us-east-1.amazonaws.com/test/streams/event-pipel
Detail about the stream

Write into this stream
http PUT https://wle4ucytwl.execute-api.us-east-1.amazonaws.com/test/streams/event-pipel/record Data="testing" PartitionKey=1
If it works you’ll see something like this with ShardID & SequenceNumber

![result](https://github.com/sammycoderu/event-pipeline/blob/master/pipeline-pics/result.png)

After 60 seconds (after batch collection) the data should be in s3.  GZIP file

https://httpie.org/docs#installation
