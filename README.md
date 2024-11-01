# Kogito and Infrastructure services

Accessing the docker-compose services:
1. __PostgreSQL database__: [http://localhost:http://localhost:8055/](http://localhost:8055/)

### Steps to reproduce

1. Build the Kogito project
   ``mvn "-Pbamoe-persistence" "-Pbamoe-jobs" "-Pcontainer" clean package``
   
2. Start the docker-compose
   ``(cd docker-compose && docker compose --profile infra up -d)``
   
3. Start the application
   ``java -jar target/quarkus-app/quarkus-run.jar``
   
4. Start a new process instance with input to specify the expiration time: \
  ``curl --location 'http://localhost:8080/TestProcess'`` \\\
  ``--header 'Accept: application/json'`` \\\
  ``--header 'Content-Type: application/json'`` \\\
  ``--data '{"expirationTime":"PT60S"}'``

5. About 30 seconds before the timer expires, stop the haproxy
  ``(cd docker-compose/ && docker compose stop haproxy)``

6. The connection to the database will be lost. This is the expected log:
  ```
2024-11-01 16:41:29,844 mweiler-ibm-com ERROR [org.kie.kogito.jobs.service.management.JobServiceInstanceManager:99] (vert.x-eventloop-thread-0) Error on heartbeat JobServiceManagementInfo[id=kogito-jobs-service-leader, lastHeartbeat=2024-11-01T22:38:04.004Z, token=9c6047ec-b292-4592-8096-a2c32a6ca903]: io.netty.channel.AbstractChannel$AnnotatedConnectException: Connection refused: localhost/127.0.0.1:5432
Caused by: java.net.ConnectException: Connection refused
	at java.base/sun.nio.ch.Net.pollConnect(Native Method)
	at java.base/sun.nio.ch.Net.pollConnectNow(Net.java:672)
	at java.base/sun.nio.ch.SocketChannelImpl.finishConnect(SocketChannelImpl.java:946)
	at io.netty.channel.socket.nio.NioSocketChannel.doFinishConnect(NioSocketChannel.java:337)
	at io.netty.channel.nio.AbstractNioChannel$AbstractNioUnsafe.finishConnect(AbstractNioChannel.java:339)

  ```

7.  When the timer is supposed to fire, you should see an error:
  ```
2024-11-01 16:49:14,892 mweiler-ibm-com INFO  [org.kie.kogito.jobs.service.job.DelegateJob:62] (vert.x-eventloop-thread-0) Executing job for context: JobDetails[id='197a2197-d5d9-49a9-85b7-8949a1b22c67', correlationId='197a2197-d5d9-49a9-85b7-8949a1b22c67', status=SCHEDULED, lastUpdate=null, retries=0, executionCounter=0, scheduledId='null', recipient=RecipientInstance{recipient=InVMRecipient [data=InVMPayloadData [data=ProcessInstanceJobDescription{id='197a2197-d5d9-49a9-85b7-8949a1b22c67', timerId=1', expirationTime=org.kie.kogito.jobs.DurationExpirationTime@5ba5d973, priority=5, processInstanceId='a15ccaa0-9f00-4dd2-9290-074b8e0ef511', rootProcessInstanceId='null', processId='TestProcess', rootProcessId='null', nodeInstanceId='54b77f73-e559-4f55-81e8-bf04118573c6'}]]}, trigger=org.kie.kogito.timer.impl.SimpleTimerTrigger@26b41ab7, executionTimeout=null, executionTimeoutUnit=null, created=null]
2024-11-01 16:49:14,893 mweiler-ibm-com DEBUG [org.kie.kogito.persistence.jdbc.JDBCProcessInstances:110] (vert.x-eventloop-thread-0) Find process instance id: a15ccaa0-9f00-4dd2-9290-074b8e0ef511, mode: READ_ONLY
2024-11-01 16:49:14,896 mweiler-ibm-com WARN  [io.agroal.pool:83] (vert.x-eventloop-thread-0) Datasource '<default>': This connection has been closed.
2024-11-01 16:49:14,901 mweiler-ibm-com ERROR [org.kie.kogito.jobs.service.job.DelegateJob:88] (executor-thread-1) Job execution error response received: JobExecutionResponse[message='Unexpected error when executing Embedded request for job: 197a2197-d5d9-49a9-85b7-8949a1b22c67. Error finding process instance a15ccaa0-9f00-4dd2-9290-074b8e0ef511', code='null', timestamp=2024-11-01T16:49:14.900312229-06:00[America/Edmonton], jobId='197a2197-d5d9-49a9-85b7-8949a1b22c67']
2024-11-01 16:49:14,907 mweiler-ibm-com ERROR [org.kie.kogito.jobs.service.scheduler.BaseTimerJobScheduler:347] (executor-thread-1) Failed to retrieve job due to Connection refused: localhost/127.0.0.1:5432
2024-11-01 16:49:14,908 mweiler-ibm-com WARN  [org.kie.kogito.jobs.service.utils.ErrorHandling:71] (executor-thread-1) Error skipped when processing input: JobExecutionResponse[message='Unexpected error when executing Embedded request for job: 197a2197-d5d9-49a9-85b7-8949a1b22c67. Error finding process instance a15ccaa0-9f00-4dd2-9290-074b8e0ef511', code='null', timestamp=2024-11-01T16:49:14.900312229-06:00[America/Edmonton], jobId='197a2197-d5d9-49a9-85b7-8949a1b22c67'].: io.netty.channel.AbstractCha
  ```

8. Start the haproxy again
   ``(cd docker-compose/ && docker compose start haproxy)``

9. Once the connection to the database has been restored, and the period timer loading job runs again, you should see the timer getting executed:
```
2024-11-01 16:50:03,276 mweiler-ibm-com INFO  [org.kie.kogito.jobs.service.scheduler.JobSchedulerManager:185] (vert.x-eventloop-thread-0) Loading jobs to schedule from the repository, fromFireTime: 2024-11-01T22:39:03.276Z[UTC] toFireTime: 2024-11-01T22:55:03.276Z[UTC].
2024-11-01 16:50:03,289 mweiler-ibm-com DEBUG [org.kie.kogito.jobs.service.scheduler.JobSchedulerManager:213] (vert.x-eventloop-thread-0) Job found, id: 197a2197-d5d9-49a9-85b7-8949a1b22c67, nextFireTime: 2024-11-01T22:49:14.888Z[UTC], created: 2024-11-01T22:48:14.941282Z[UTC], status: SCHEDULED is overdue and will be rescheduled
2024-11-01 16:50:04,327 mweiler-ibm-com DEBUG [org.jbpm.workflow.instance.impl.WorkflowProcessInstanceImpl:660] (executor-thread-1) Signal timerTriggered received with data TimerInstance [id=197a2197-d5d9-49a9-85b7-8949a1b22c67, timerId=1, delay=0, period=0, activated=null, lastTriggered=null, processInstanceId=null] in process instance a15ccaa0-9f00-4dd2-9290-074b8e0ef511
2024-11-01 16:50:04,337 mweiler-ibm-com DEBUG [org.jbpm.workflow.instance.impl.WorkflowProcessInstanceImpl:660] (executor-thread-1) Signal workItemTransition received with data DefaultWorkItemTransitionImpl [id=abort, data={}, policies=[], termination=ABORT] in process instance a15ccaa0-9f00-4dd2-9290-074b8e0ef511
Process instance expired
```

10. Using graphql, check that the process has ended successfully.

