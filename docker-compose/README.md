# Kogito and Infrastructure services

Accessing the docker-compose services:
1. __Kogito Management Console__: [http://localhost:8280/](http://localhost:8280/) (user:jdoe, password:jdoe)
2. __Kogito Tasks Console__: [http://localhost:8380/](http://localhost:8380/) (user:jdoe, password:jdoe)
3. __PostgreSQL database__: [http://localhost:http://localhost:8055/](http://localhost:8055/)

### Steps to reproduce

1. Build the Kogito project
   ``mvn "-Pbamoe-persistence" "-Pbamoe-jobs" clean package``
   
2. Build the Kogito Docker image
   ``docker build -t dev.local/kogito-bpmn-pgsql-ha-bpmn-workflow:1.0.0 -f .\src\main\docker\Dockerfile.jvm .``
   
3. Start the docker-compose
   ``cd kogito-bpmn-pgsql-ha-bpmn-workflow\docker-compose && docker-compose up -d``
   
4. Identify the Postgres primary instance by this log:
   ``INFO: no action. I am (postgresql<0 or 1>), the leader with the lock``

5. Identify the Postgres secondary instance by this log:
   ``INFO: no action. I am (postgresql<0 or 1>), a secondary, and following a leader (postgresql<0 or 1>)``

6. Start a new process instance with input to specify the expiration time: \
  ``curl --location 'http://localhost:8080/TestProcess'`` \\\
  ``--header 'Accept: application/json'`` \\\
  ``--header 'Content-Type: application/json'`` \\\
  ``--data '{"expirationTime":"PT120S"}'``

7. About 30 seconds before the timer expires, stop the container of the Postgres secondary instance
  ``docker-compose stop pg-master``  or ``docker-compose stop pg-slave``

8. The promote of secondary to primary will take place. It will take over 30-45 seconds. This is the expected log:
  ```
  2024-10-19 09:50:44 2024-10-19 07:50:44.600 UTC [77] FATAL:  could not receive data from WAL stream: server closed the connection unexpectedly
  2024-10-19 09:50:44             This probably means the server terminated abnormally
  2024-10-19 09:50:44             before or while processing the request.
  2024-10-19 09:50:44 2024-10-19 07:50:44.600 UTC [40] LOG:  invalid record length at 0/6007978: expected at least 24, got 0
  2024-10-19 09:50:44 2024-10-19 07:50:44.605 UTC [2874] FATAL:  could not connect to the primary server: connection to server at "pg-master" (172.19.0.4), port 5432 failed: Connection refused
  2024-10-19 09:50:44             Is the server running on that host and accepting TCP/IP connections?
  2024-10-19 09:50:44 2024-10-19 07:50:44.605 UTC [40] LOG:  waiting for WAL to become available at 0/6007990
  2024-10-19 09:50:46 2024-10-19 07:50:46.316 UTC [38] LOG:  restartpoint starting: time
  2024-10-19 09:50:47 2024-10-19 07:50:47.280 UTC [38] LOG:  restartpoint complete: wrote 10 buffers (0.1%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.909 s, sync=0.021 s, total=0.964 s; sync files=9, longest=0.015 s, average=0.003 s; distance=29 kB, estimate=26376 kB; lsn=0/60078C8, redo lsn=0/6007890
  2024-10-19 09:50:47 2024-10-19 07:50:47.280 UTC [38] LOG:  recovery restart point at 0/6007890
  2024-10-19 09:50:47 2024-10-19 07:50:47.280 UTC [38] DETAIL:  Last completed transaction was at log time 2024-10-19 07:47:52.211088+00.
  2024-10-19 09:30:30 localhost:5432 - rejecting connections
  2024-10-19 09:30:30 localhost:5432 - rejecting connections
  2024-10-19 09:30:31 localhost:5432 - rejecting connections
  2024-10-19 09:30:32 localhost:5432 - accepting connections
  2024-10-19 09:50:52 2024-10-19 07:50:52,939 INFO: no action. I am (postgresql1), a secondary, and following a leader (postgresql0)
  2024-10-19 09:50:52 2024-10-19 07:50:52.977 UTC [2889] FATAL:  could not connect to the primary server: could not translate host name "pg-master" to address: Name or service not known
  2024-10-19 09:50:52 2024-10-19 07:50:52.978 UTC [40] LOG:  waiting for WAL to become available at 0/6007990
  2024-10-19 09:50:57 2024-10-19 07:50:57.893 UTC [2904] FATAL:  could not connect to the primary server: could not translate host name "pg-master" to address: Name or service not known
  2024-10-19 09:50:57 2024-10-19 07:50:57.893 UTC [40] LOG:  waiting for WAL to become available at 0/6007990
  2024-10-19 09:51:02 2024-10-19 07:51:02,937 INFO: no action. I am (postgresql1), a secondary, and following a leader (postgresql0)
  2024-10-19 09:51:02 2024-10-19 07:51:02.991 UTC [2918] FATAL:  could not connect to the primary server: could not translate host name "pg-master" to address: Name or service not known
  2024-10-19 09:51:02 2024-10-19 07:51:02.991 UTC [40] LOG:  waiting for WAL to become available at 0/6007990
  2024-10-19 09:51:07 2024-10-19 07:51:07.908 UTC [2933] FATAL:  could not connect to the primary server: could not translate host name "pg-master" to address: Name or service not known
  2024-10-19 09:51:07 2024-10-19 07:51:07.908 UTC [40] LOG:  waiting for WAL to become available at 0/6007990
  2024-10-19 09:51:12 2024-10-19 07:51:12,937 INFO: no action. I am (postgresql1), a secondary, and following a leader (postgresql0)
  2024-10-19 09:51:12 2024-10-19 07:51:12.986 UTC [2948] FATAL:  could not connect to the primary server: could not translate host name "pg-master" to address: Name or service not known
  2024-10-19 09:51:12 2024-10-19 07:51:12.986 UTC [40] LOG:  waiting for WAL to become available at 0/6007990
  2024-10-19 09:51:17 2024-10-19 07:51:17,712 WARNING: Request failed to postgresql0: GET http://pg-master:8008/patroni (HTTPConnectionPool(host='pg-master', port=8008): Max retries exceeded with url: /patroni (Caused by NameResolutionError("<urllib3.connection.HTTPConnection object at 0x7f240152bad0>: Failed to resolve 'pg-master' ([Errno -2] Name or service not known)")))
  2024-10-19 09:51:17 2024-10-19 07:51:17,734 WARNING: Could not activate Linux watchdog device: Can't open watchdog device: [Errno 2] No such file or directory: '/dev/watchdog'
  2024-10-19 09:51:17 2024-10-19 07:51:17,742 INFO: promoted self to leader by acquiring session lock
  2024-10-19 09:51:17 2024-10-19 07:51:17.744 UTC [40] LOG:  received promote request
  2024-10-19 09:51:17 server promoting
  2024-10-19 09:51:17 2024-10-19 07:51:17.914 UTC [2966] FATAL:  could not connect to the primary server: could not translate host name "pg-master" to address: Name or service not known
  2024-10-19 09:51:17 2024-10-19 07:51:17.914 UTC [40] LOG:  waiting for WAL to become available at 0/6007990
  2024-10-19 09:51:17 2024-10-19 07:51:17.914 UTC [40] LOG:  redo done at 0/6007940 system usage: CPU: user: 0.00 s, system: 0.00 s, elapsed: 1245.98 s
  2024-10-19 09:51:17 2024-10-19 07:51:17.914 UTC [40] LOG:  last completed transaction was at log time 2024-10-19 07:47:52.211088+00
  2024-10-19 09:51:17 2024-10-19 07:51:17.926 UTC [40] LOG:  selected new timeline ID: 3
  2024-10-19 09:51:17 2024-10-19 07:51:17.986 UTC [40] LOG:  archive recovery complete
  2024-10-19 09:51:18 2024-10-19 07:51:18.003 UTC [38] LOG:  checkpoint starting: force
  2024-10-19 09:51:18 2024-10-19 07:51:18.006 UTC [36] LOG:  database system is ready to accept connections
  2024-10-19 09:51:18 2024-10-19 07:51:18.075 UTC [38] LOG:  checkpoint complete: wrote 2 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.019 s, sync=0.005 s, total=0.073 s; sync files=2, longest=0.003 s, average=0.003 s; distance=0 kB, estimate=23738 kB; lsn=0/60079E0, redo lsn=0/60079A8
  2024-10-19 09:51:18 2024-10-19 07:51:18,755 INFO: Lock owner: postgresql1; I am postgresql1
  2024-10-19 09:51:18 2024-10-19 07:51:18,770 INFO: Reaped pid=2983, exit status=0
  2024-10-19 09:51:18 2024-10-19 07:51:18,777 INFO: no action. I am (postgresql1), the leader with the lock
  2024-10-19 09:51:28 2024-10-19 07:51:28,753 INFO: Lock owner: postgresql1; I am postgresql1
  2024-10-19 09:51:28 2024-10-19 07:51:28,772 INFO: Dropped unknown replication slot 'postgresql0'
  2024-10-19 09:51:28 2024-10-19 07:51:28,776 INFO: no action. I am (postgresql1), the leader with the lock
  2024-10-19 09:51:38 2024-10-19 07:51:38,762 INFO: no action. I am (postgresql1), the leader with the lock
  ```

9.  While promoting, the timer will trigger in the Kogito app container. It will fail and then it will unschedule.


### Issue noticed
The timer is unscheduled, and it won't be triggered again. It won't even trigger when it's manually rescheduled either.