# Docker Test

Create a Test Docker application with Docker Compose that include the following services:

- 1 Redis container
- 2 Apache containers
    - serve one index.php file that
   	-- outputs a json containing the current date and the IP of the user
   	-- sets a session variable called `count` that is incremented on each visit
    - save all session information in the Redis container
    - defer all interpreting of PHP code to the PHP-FPM container
    - have error logging and access logging enabled
    - logs need to be sent to the Logstash container
- 1 PHP-FPM container
    - PHP7.1
    - used for interpreting PHP code from the Apache containers
- 1 Load Balancer for the Apache containers
- 1 Jenkins container
    - has a job that runs once per day and deletes indexes older than 30 days from Elasticsearch
- 1 Elasticsearch container
- 1 Kibana container
    - allows viewing the data from the Elasticsearch container
- 1 Logstash container
    - ingests access and error logs from the Apache containers and saves to Elasticsearch

# Getting started

- *loadbalancer: port 8080* (eg: http://localhost:8080). To see the index.php
- *kibana: port 5601* (eg: http://localhost:5601). Kibana UI to see a chart with logging events.
- *elasticsearch api: port 9200* (eg: http://localhost:9200/_search?pretty).
- *elasticsearch transport: port 9300*. Elastic transport is listening on elk:9300
- *logstash: port 5044*. Where logstash is listening for logging events to store
- *rebrow: port 5001*. Redis web UI to explore the stored key/values

To get logged in to a container's shell
```
docker exec -t -i container_name /bin/bash
```

### Testing

Go to http://localhost:8080. You should get a JSON. Refresh the page multiple times; the counter should increase at each refresh.

To check the value stored in Redis open http://localhost:5001 and use `redis` as hostname.


## Logging

```
# Start filebeat and apache
# Not best practice, there should be a separate filebeat container with the logs mounted as volumes but it's overkill for a demo
CMD /etc/init.d/filebeat start && \
    /usr/sbin/apache2ctl -D FOREGROUND
```

- Created a `filebeat.yml` config file that is copied into the `ẁeb` container at build time. In this config file I set the input files path and linked the output to the logstash endpoint that is reachable in the composer network through the "elk" hostname (see snippet below). 

```
filebeat:
  # List of prospectors to fetch data.
  prospectors:
      paths:
        - /var/log/apache2/access.log
        - /var/log/apache2/error.log

output:
  logstash:
    enabled: true
    hosts:
      - elk:5044
    timeout: 15
```

### Testing

Go to http://localhost:9090, access with username `admin` and password `admin` and run the `elastic-index-cleanup` job. Check if the console output is similar to the one reported above. Out of the box it deletes only indexes older than 30 days. If you want to delete the index that was just created at cluster startup, change to '-1' the value of 
```
OLDERTHAN=30 # Delete indexes older than N days
```
and re-run the job.