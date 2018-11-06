# Errata for the course:

- `docker service` commands no longer run detached by default as of `17.10`
  - This is nice because you get a **progress indicator** in the new interactive mode
  - If you want to run detached simply pass `--detach`
  - Applies to `service create`, `service update`, `service rollback`, and `service scale`
  - For command examples before and after this change and a history of the change, check out this PR, it's great: [docker/cli#525](https://github.com/docker/cli/pull/525)
  
Updated as of 17.10

Keep an eye out for future changes, this is the easiest spot for me to update.




============================
Notes on Docker Swarm course
============================

Start a singlr container running a web app:

  docker run --rm -d -p 3000:3000 swarmgs/nodewebstress
test by:
  http://192.168.209.101:3000/customer%20/1         <-- where 192.168.209.101 is the IP of the docker host

We can use apach tool "ab" (apache bench) to performance test this:
  yum install apache2-utils
  yum install httpd
  
  ab -n 100 http://192.168.209.101:3000/customer%20/1       <-- use apache bench to send 100 requests to our API
  
    Benchmarking 192.168.209.101 (be patient).....done
    Server Software:        
    Server Hostname:        192.168.209.101
    Server Port:            3000
    Document Path:          /customer%20/1
    Document Length:        144 bytes
    Concurrency Level:      1
    Time taken for tests:   1.580 seconds
    Complete requests:      100
    Failed requests:        0
    Write errors:           0
    Non-2xx responses:      100
    Total transferred:      38800 bytes
    HTML transferred:       14400 bytes
    Requests per second:    63.27 [#/sec] (mean)
    Time per request:       15.805 [ms] (mean)
    Time per request:       15.805 [ms] (mean, across all concurrent requests)
    Transfer rate:          23.97 [Kbytes/sec] received

Showing 63 requests per seconds - lets try with 1000 requests:


  ab -n 1000 http://192.168.209.101:3000/customer%20/1  
    Requests per second:    683.06 [#/sec] (mean)

Ramp it up to 50,000 requests:
  ab -n 50000 http://192.168.209.101:3000/customer%20/1
    Requests per second:    1463.39 [#/sec] (mean)
    Requests per second:    1615.13 [#/sec] (mean)
    Requests per second:    1335.36 [#/sec] (mean)
    Requests per second:    1573.60 [#/sec] (mean)
    
Test with a concurrency of 10:
  ab -n 50000 -c 10 http://192.168.209.101:3000/customer%20/1
    Requests per second:    1679.58 [#/sec] (mean)
    Requests per second:    1911.75 [#/sec] (mean)
    
Test with a concurrency of 100:
  ab -n 50000 -c 100 http://192.168.209.101:3000/customer%20/1
    Requests per second:    1316.95 [#/sec] (mean)

So we seem to have hit a max of around 1600 requests per second. Assuming our API is as efficient as it can be how can we increase this value - We could try adding more containers (assuming we have enough CPU, mem etc on the host machine).

So scaling in production is an issue which needs consideration. 
Other issues are:

  Scaling Capacity
    Load balancing
    Adding containers
  Container Failure
    Restarting containers
  Node Failure
    Redistribute Containers
    Restart Node
    Placement
    Node maintenance
  Internal Communication
    container dependencies
  
Lets specify a network and run a container on it
  docker network create -d=bridge company
  docker network ls
    NETWORK ID          NAME                DRIVER              SCOPE
    299c8a956260        bridge              bridge              local
    2173055a4236        company             bridge              local

Now we can run a container on that network:
  docker run --network company --rm -d --name customer-api swarmgs/customer
  
  docker ps
  CONTAINER ID        IMAGE                   COMMAND             CREATED             STATUS              PORTS                    NAMES
  63d8f440fe20        swarmgs/customer        "npm start"         7 seconds ago       Up 6 seconds        3000/tcp               company

We can run another API that talks to the customer-api container - docker supports service discovery so as customer-api has a name and is listening on port 3000 - we can specify:

  docker run --network company --rm -d --name balance-api -p 4000:3000 -e=MYWEB_CUSTOMER_API=customer-api:3000 swarmgs/balance

  docker ps
  CONTAINER ID        IMAGE               COMMAND         CREATED              STATUS              PORTS                    NAMES
  e50577f3bdf1        swarmgs/balance     "npm start"     4 seconds ago        Up 1 second         0.0.0.0:4000->3000/tcp   balance-api
  2036e9983949        swarmgs/customer    "npm start"     About a minute ago   Up About a minute   3000/tcp                 customer-api

We can test the balance-api on port 4000:
  http://192.168.209.101:4000/balance/1
and if we stop customer-api we observe that balance-api also stops working


===============
Docker-Compose
===============
We can simplify all of these steps using docker compose:


   
    
  
