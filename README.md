# Kafka HTTP Sink Connector

The HTTP sink connector allows you to listen to topic(s) and send the data to any HTTP API.

### Installing Connector

1. Download / build jar
2. Copy the jar to the desired location. For example, you can create a directory named `<path-to-kafka>/share/kafka/plugins` then copy the connector plugin jar.
3. Add this to the plugin path in your Connect properties file. For example, `plugin.path=/usr/local/share/kafka/plugins`. Kafka Connect finds the plugins using its plugin path. A plugin path is a comma-separated list of directories defined in the Kafka Connect's worker configuration.
4. Start the Connect workers with that configuration. Connect will discover all connectors defined within those plugins.

### Connector configuration properties

Property | Description | Value |Default|Required|  
---|---|---|---|:---:|
topics | Topic(s) you want to listen to | String (use comma to pass multiple topics)| |Y|
callback.request.url| Callback url that should be called when a message is received on the topic | String | |Y
callback.request.methods| Method of the callback url| POST, PUT,DELETE | | Y|
callback.request.headers| Headers to be passed to the callback|String (use separator to pass multiple values. Default separator is pipe '&#124;')| |N
callback.header.separator| Separator character used to separate header properties| Character| &#124; |Y
callback.timeout.connect.ms	| Connect timeout in ms when connecting to callback url.|Integer|60000|Y
callback.timeout.read.ms	| Read timeout in ms when reading response from call back url.|Integer|60000|Y
exception.strategy| Strategy to be used in-case the callback request returns response with `retry` flag `true`.| PROGRESS_BACK_OFF_DROP_MESSAGE, PROGRESS_BACK_OFF_STOP_TASK, DROP_MESSAGE, STOP_TASK | PROGRESS_BACK_OFF_DROP_MESSAGE | Y
retry.backoff.sec| The time in seconds to wait following an error before a retry attempt is made.| Comma separated Integers. Total of all retries should not exceed offset.flush.interval.ms set in connect-distributed.properties | 5,30,60,300,600 | Y

### Exception strategies

> The exception strategy is triggered if the response send back by the callback url had a retry flag set to true.
  <br> Sample Response: Retry **not** triggered
  ```json
    {
      "message":"Unable to process. An error happened: Identity - User with ID=test not found",
      "retry":false
    }
  ```
> Sample Response:  **Retry triggered**
  ```json
    {
      "message":"Unable to process. An unhandled exception has been thrown.",
      "retry":true
    }
  ```

One can opt for one of the below exception strategies to retry the failed callback request.
  
  
- PROGRESS_BACK_OFF_DROP_MESSAGE :
  <br>Retry request with increasing amount of wait time between request and DROP the message if not successful.  
  <br>The request is retied after waiting for a certain amount of time as configured in the `retry.backoff.sec` property.
  The reties generally configured to wait for increased amount of time after each failure. 
  So with `retry.backoff.sec = 5,30,60,300,600` the PROGRESS_BACK_OFF_DROP_MESSAGE will try to call the callback request url 
  after 5sec, 30sec, 1min, 5min and 10mins between each failed try and ultimately drop the message and move to processing the next message in que.
  <br> This is the default and recommended strategy. 

- PROGRESS_BACK_OFF_STOP_TASK:
  <br> Retry request with increasing amount of wait time between request and STOP the connector if not successful. 
  `The connector would have to be manually restarted so that it starts to consume messages again.`   
  <br>The request is retied after waiting for a certain amount of time as configured in the `retry.backoff.sec` property.
  The reties generally configured to wait for increased amount of time after each failure. 
  So with `retry.backoff.sec = 5,30,60,300,600` the PROGRESS_BACK_OFF_STOP_TASK will try to call the callback request url 
  after 5sec, 30sec, 1min, 5min and 10mins between each failed try and ultimately stop the connector and **no new messages in the que would be consumed.**
  <br> Use this strategy if you do not want to loose any message.The application is continuously monitored for any errors and connector restart can be manually triggered.  
  
- DROP_MESSAGE:
  <br> Drop the message and move to processing the next message in que. Failures from the callback request are ignored.
  <br> Use this strategy if you do not care about loosing messages.   
  
- STOP_TASK:  
  <br> Immediately stop the connector. **No new messages in the que would be consumed.** `The connector would have to be manually restarted so that it starts to consume messages again.`
  <br> Use this strategy if you do not want to loose any message. The application is continuously monitored for any errors and connector restart can be manually triggered.
     
### Kafka connect rest API

- Create new connection
    - Endpoint: /connectors
    - Method: POST
    - Payload:
    ```json
        {
            "name": "<Integration project name>",
            "config": {
                "connector.class":"nz.ac.auckland.kafka.http.sink.HttpSinkConnector",
                "tasks.max":"1",                
                "value.converter":"org.apache.kafka.connect.storage.StringConverter",
              	"key.converter":"org.apache.kafka.connect.storage.StringConverter",
              	"header.converter":"org.apache.kafka.connect.storage.StringConverter",            	
          	    "topics":"<Topics to listen to>",
                "callback.request.url":"<Callback url to call>",
                "callback.request.method":"<Callback url request method>",
                "callback.request.headers":"<Callback url headers>",
                "retry.backoff.sec":"5,60,120,300,600",
                "exception.strategy":"PROGRESS_BACK_OFF_DROP_MESSAGE"
            }
        }
    ```
- Update existing connection
    - Endpoint: /connectors/<connection name(Integration project name)>
    - Method: PUT
    - Payload:
    ```json
        {
            "connector.class":"nz.ac.auckland.kafka.http.sink.HttpSinkConnector",
            "tasks.max":"1",                
            "value.converter":"org.apache.kafka.connect.storage.StringConverter",
            "key.converter":"org.apache.kafka.connect.storage.StringConverter",
            "header.converter":"org.apache.kafka.connect.storage.StringConverter",            	
            "topics":"<Topics to listen to>",
            "callback.request.url":"<Callback url to call>",
            "callback.request.method":"<Callback url request method>",
            "callback.request.headers":"<Callback url headers>",
            "retry.backoff.sec":"5,60,120,300,600",
            "exception.strategy":"PROGRESS_BACK_OFF_DROP_MESSAGE"
        }
    ```    
    
- List connections:
    - Endpoint: /connectors
    - Method: GET    

- Connection Details:
    - Endpoint: /connectors/<connection name(Integration project name)>
    - Method: GET

- Pause Connection: Pause consuming messages. When resumed connection will start consuming messages from the offset it was paused at.
    - Endpoint: /connectors/<connection name(Integration project name)>/pause
    - Method: GET        

- Resume Connection: Resume paused connection. The connection will start consuming messages from the offset it was paused at.
    - Endpoint: /connectors/<connection name(Integration project name)>/resume
    - Method: GET

- Delete Connection: Connection would be deleted and cannot be restored back.
    - Endpoint: /connectors/<connection name(Integration project name)>
    - Method: DELETE
    

### References

- [Installing and Configuring Kafka Connect](https://docs.confluent.io/current/connect/userguide.html#connect-configuring-workers)             