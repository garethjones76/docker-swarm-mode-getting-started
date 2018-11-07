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
First lets install docker-compose:

  sudo yum -y install python-pip
  sudo pip install --upgrade pip
  sudo pip install docker-compose
  sudo chmod +x /usr/bin/docker-compose
  sudo cp /usr/bin/docker-compose /usr/local/bin/
  docker-compose --version

Now create a compose file:
  mkdir load-test
  cd load-test
  
  vi company.yml:
    version: '2'

    services:
      customer-api:
          image: swarmgs/customer
      balance-api:
          image: swarmgs/balance
         ports:
             - 4000:3000
         environment:
             MYWEB_CUSTOMER_API: CUSTOMER-API:3000

Now start the docker service defined by the compose file:
  docker-compose -f company.yml up -d        <-- start service defined in company.yml in detached mode
    Creating network "load-test_default" with the default driver
    Creating load-test_balance-api_1  ... done
    Creating load-test_customer-api_1 ... done

We can see a network was created:
  docker network ls
  NETWORK ID          NAME                DRIVER              SCOPE
  ...
  273c949f6295        load-test_default   bridge              local
  
And we can hit the service as before:
  http://192.168.209.101:4000/balance/1

We can stop/start the composed services individually:
  docker-compose -f company.yml stop customer-api
  docker-compose -f company.yml start customer-api

==============
Docker Swarm
==============

So in a more prodcution-like environment we will need multi-node definitions. This is where docker swarm comes in:

Lets start swarm mode:

  docker swarm init
  Swarm initialized: current node (y8y0p9uq59q1gc81449afjezh) is now a manager.

  To add a worker to this swarm, run the following command:

      docker swarm join \
     --token SWMTKN-1-696rnc8wu8l9qpvwow3lbi1p33cnnkpd2h09i6eeof16bu4k3x-36wlouf3cegmla2h8b1gdsr1i \
     192.168.209.101:2377

  To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
  
we can see the current status:
  docker node ls
    ID                           HOSTNAME         STATUS  AVAILABILITY  MANAGER STATUS
    y8y0p9uq59q1gc81449afjezh *  centos1.gjj.com  Ready   Active        Leader
  
  docker node inspect self
    [
    {
        "ID": "y8y0p9uq59q1gc81449afjezh",
        "Version": {
            "Index": 9
        },
        "CreatedAt": "2018-11-06T16:27:19.59108444Z",
        "UpdatedAt": "2018-11-06T16:27:20.20991763Z",
        "Spec": {
            "Role": "manager",
            "Availability": "active"
        },
        "Description": {
            "Hostname": "centos1.gjj.com",
            "Platform": {
                "Architecture": "x86_64",
                "OS": "linux"
    ...
    ...

==========
Services
==========
A service in swarm mode is similar to "docker run" in single container mode. So if we wanted to create a service for nginx in swarm mode we would do:
  docker service create --name web -p 8080:80 nginx
  
We can see our service is running:
  docker service ls
    ID            NAME  MODE        REPLICAS  IMAGE
    nm3xmlnrj013  web   replicated  1/1       nginx:latest
And we can see our container is running:
  docker ps
    CONTAINER ID        IMAGE                       COMMAND                 CREATED         STATUS         PORTS           NAMES
    98064d2c6f14        nginx@sha256:b73f527d86e34  "nginx -g 'daemon ..."  6 seconds ago   Up 4 seconds   80/tcp web.1.foztattmsfrlavde

BUT we now get service discovery, load balancing, secrets definition, encryption, certificate creation and rotation etc

So - lets demonstrate this by scaling up to 2 containers:

  docker service update --replicas=2 web

And we now have 2 containers running:
  docker service ps web
    ID            NAME   IMAGE         NODE             DESIRED STATE  CURRENT STATE           ERROR  PORTS
    foztattmsfrl  web.1  nginx:latest  centos1.gjj.com  Running        Running 13 minutes ago         
    i8h7pfwhnqki  web.2  nginx:latest  centos1.gjj.com  Running        Running 7 seconds ago    
    

Alternatively we can use the "scale" command:

  docker service scale web=4
    web scaled to 4
    
And we now have 4 containers running:
  docker service ps web
    ID            NAME   IMAGE         NODE             DESIRED STATE  CURRENT STATE           ERROR  PORTS
    foztattmsfrl  web.1  nginx:latest  centos1.gjj.com  Running        Running 18 minutes ago         
    i8h7pfwhnqki  web.2  nginx:latest  centos1.gjj.com  Running        Running 4 minutes ago          
    wfdsbbpud94o  web.3  nginx:latest  centos1.gjj.com  Running        Running 13 seconds ago         
    phso3h6r4mkj  web.4  nginx:latest  centos1.gjj.com  Running        Running 13 seconds ago   

Thw swarm manager continually monitors the swarm and repairs problems if discovered. For example - we've defined a service with 4 containers. If we kill a contaniner - another will be started:
    docker ps
    CONTAINER ID        IMAGE     
    dd1173593eee        nginx@sha256:b73f527d86e3461fd652f62cf47e7b375196063bbbd503e853af5be16597cb2e   "nginx -g 'daemon ..."   6 
    bf677817a0f7        nginx@sha256:b73f527d86e3461fd652f62cf47e7b375196063bbbd503e853af5be16597cb2e   "nginx -g 'daemon ..."   6 
    
    docker stop dd11
    
    docker service ps web
    ID            NAME       IMAGE         NODE             DESIRED STATE  CURRENT STATE                ERROR  PORTS
    foztattmsfrl  web.1      nginx:latest  centos1.gjj.com  Running        Running 26 minutes ago              
    i8h7pfwhnqki  web.2      nginx:latest  centos1.gjj.com  Running        Running 12 minutes ago              
    wfdsbbpud94o  web.3      nginx:latest  centos1.gjj.com  Running        Running 8 minutes ago               
    fzxuyqtnz4gz  web.4      nginx:latest  centos1.gjj.com  Running        Running about a minute ago          
    phso3h6r4mkj   \_ web.4  nginx:latest  centos1.gjj.com  Shutdown       Complete about a minute ago 
    
We still have 4 containers running as the swarm manager has started a new task (container) to maintain our service at the specified scale of 4

So - going back to our original test case - we can create it as a single node service:
  docker service create --name customer-api -p 3000:3000 swarmgs/customer
  docker service ls
      ID            NAME          MODE        REPLICAS  IMAGE
    nm3xmlnrj013  web           replicated  4/4       nginx:latest
    x0yb80z3vksm  customer-api  replicated  1/1       swarmgs/customer:latest
    
Then test with apache bench:
  ab -n 50000 -c 100 http://192.168.209.101:3000/customer%20/1
    Requests per second:    151.17 [#/sec] (mean)
    Requests per second:    125.25 [#/sec] (mean)

So now lets scale it up :
  docker service scale customer-api=2
    customer-api scaled to 2
  docker service ps customer-api
    ID            NAME            IMAGE                    NODE             DESIRED STATE  CURRENT STATE           ERROR  PORTS   
    hlxgx60bowkw  customer-api.1  swarmgs/customer:latest  centos1.gjj.com  Running        Running 15 minutes ago         
    qfzys6tat94e  customer-api.2  swarmgs/customer:latest    centos1.gjj.com  Running        Running 40 seconds ago
 
Then test with apache bench:
  ab -n 50000 -c 100 http://192.168.209.101:3000/customer%20/1
    Requests per second:    150.41 [#/sec] (mean)
    Requests per second:    131.08 [#/sec] (mean)
    Requests per second:    146.71 [#/sec] (mean)

Lets look at scaling across additional nodes

First lets start a manger in swarm mode (we first need to leave swarm mode from the previous excercises)
  docker swarm leave --force

The IP for our VM is 192.168.209.101 - so lets set that as the advertised address:
  docker swarm init --advertise-addr 192.168.209.101
    Swarm initialized: current node (ypis1ql0595wp5hq7e3ex36yq) is now a manager.

    To add a worker to this swarm, run the following command:

      docker swarm join \
      --token SWMTKN-1-46tsv11c9lahuwi53tlez2qqsdmp1p166b3j7vsw082pcuy8hn-c3s511zzhylg2e2ws0jw88xvt \
      192.168.209.101:2377

    To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
This node is both a manager node and (be default) a worker node. It is also the leader - ie the active manager node. The swarm uses the RAFT algorthm to decide where to send requests etc. The node creates a distributed data store which is replicated to the manager nodes to keep track of the swarm services etc and also becomes a CA (cert Authority) and issues certs to each of the joining worker/manager nodes (eg above -  --token SWMTKN-1-46tsv11c9lahuwi53tlez2qqsdmp1p166b3j7vsw082pcuy8hn-c3s511zzhylg2e2ws0jw88xvt)

These tokens can be rotated using 
  docker swarm join-token --rotate

To add a new worker node copy the string above;
  docker swarm join \
      --token SWMTKN-1-46tsv11c9lahuwi53tlez2qqsdmp1p166b3j7vsw082pcuy8hn-c3s511zzhylg2e2ws0jw88xvt \
      192.168.209.101:2377
Switch to a new node and paste this into the command line.
You should see "This node joined a swarm as a worker"

To get the command/token to join as a worker or manager use the following commands:

  docker swarm join-token worker
    To add a worker to this swarm, run the following command:

      docker swarm join \
      --token SWMTKN-1-46tsv11c9lahuwi53tlez2qqsdmp1p166b3j7vsw082pcuy8hn-c3s511zzhylg2e2ws0jw88xvt \
      192.168.209.101:2377

  docker swarm join-token manager
    To add a manager to this swarm, run the following command:
      docker swarm join \
      --token SWMTKN-1-46tsv11c9lahuwi53tlez2qqsdmp1p166b3j7vsw082pcuy8hn-a8kr3euvto0e395zwmyhvof2t \
      192.168.209.101:2377

=========================
Creating Services
=========================
First service we will create is for a  useful tool called "Service Visualizer" - available on Dockerhub at dockersamples/visualizer

To create this service (as per the readme for visualizer) enter:
  docker service create \
  --name=viz \
  --publish=8090:8080/tcp \
  --constraint=node.role==manager \
  --mount=type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock \
  dockersamples/visualizer

Here we create a service named viz using the dockersamples/visualizer image, make it available on port 8090 (mapped internally to 8080), run it on manager nodes only and lastly make the docker socket file available to the srvice as a mount

We can see the status of the service: 
  docker service ls
      ID            NAME  MODE        REPLICAS  IMAGE
    fc3sfdzdcnyt  viz   replicated  1/1       dockersamples/visualizer:latest
  
  docker service inspect viz
    <details of the service definition>
  
  docker service ps viz
    ID            NAME   IMAGE                            NODE             DESIRED STATE  CURRENT STATE          ERROR  PORTS
    mbexkhr0pegr  viz.1  dockersamples/visualizer:latest  centos4.gjj.com  Running        Running 9 seconds ago         

As we can see above the service is running on centos4.gjj.com. 
However - because of swarm service discovery - we can reach the service on any of the nodes - so we should be able to point a browser at any of the IPs with port 8090 and still hit the service:
  http://192.168.209.101:8090
  http://192.168.209.100:8090
  http://192.168.209.102:8090
  http://192.168.209.103:8090
(and that works)

If we now recreate the customer-api service - we can see it in the visualizer:
  docker service create --name customer-api -p 3000:3000 swarmgs/customer

And if we scale we can see the scaling happening in the visualizer
  docker service scale customer-api=3
The first time we do this ths process needs to download images on each node so is slow - but next time we scale it is very fast:
  docker service scale customer-api=12
  
We can move the work off a node by using the drain option
  docker node update --availability=drain 
For example we can see via the visualiser or the docker node ps" command that the customer-api app is running on centos3:
  docker node ls
    ID                           HOSTNAME         STATUS  AVAILABILITY  MANAGER STATUS
    296e07c8758frnwxq00c40grd    centos2.gjj.com  Ready   Active        
    90pw0m079vl406jzn3ywu1651    centos4.gjj.com  Ready   Active        Leader
    adyx5cnthqp5b2igb23lwiz6x    centos3.gjj.com  Ready   Active        
    ypis1ql0595wp5hq7e3ex36yq *  centos1.gjj.com  Ready   Active        Reachable
    
  docker node ps adyx5cnthqp5b2igb23lwiz6x
    ID            NAME             IMAGE                    NODE             DESIRED STATE  CURRENT STATE           ERROR  PORTS
    od2nz0tcn1wf  customer-api.10  swarmgs/customer:latest  centos3.gjj.com  Running        Running 29 minutes ago         
    yb82gn3qzy0v  customer-api.12  swarmgs/customer:latest  centos3.gjj.com  Running        Running 29 minutes ago

So we can move the service off centos3 by doing:
    docker node update --availability=drain adyx5cnthqp5b2igb23lwiz6x
    
Node centos3 now shows:
    docker node ps adyx5cnthqp5b2igb23lwiz6x
    ID            NAME             IMAGE                    NODE             DESIRED STATE  CURRENT STATE           ERROR  PORTS
    od2nz0tcn1wf  customer-api.10  swarmgs/customer:latest  centos3.gjj.com  Shutdown       Shutdown 3 seconds ago         
    yb82gn3qzy0v  customer-api.12  swarmgs/customer:latest  centos3.gjj.com  Shutdown       Shutdown 5 seconds ago   

We can check where it is running:
  docker node ps $(docker node ls -q)|grep -i running|grep -i customer
  ID            NAME             IMAGE                            NODE             DESIRED STATE  CURRENT STATE            ERROR           xdafpe4mg28r  customer-api.10      swarmgs/customer:latest          centos4.gjj.com  Running        Running about a minute ago           xdafpe4mg28r   \_ customer-api.10  swarmgs/customer:latest          centos4.gjj.com  Running        Running about a minute ago           xdafpe4mg28r   \_ customer-api.10  swarmgs/customer:latest          centos4.gjj.com  Running        Running about a minute ago           nbwnnzk8wija  customer-api.12      swarmgs/customer:latest          centos4.gjj.com  Running        Running about a minute ago           nbwnnzk8wija   \_ customer-api.12  swarmgs/customer:latest          centos4.gjj.com  Running        Running about a minute ago           nbwnnzk8wija   \_ customer-api.12  swarmgs/customer:latest          centos4.gjj.com  Running        Running about a minute ago
  
 So it is now running on centos4 - which can be confirmed by vizualizer
 
We can use the "constraint" to specify various contraints as to where we want our service to run - contraints can be identified using the "docker node inspect" command eg "Architecture" or "OS", CPUs or Memeory, or we can add labels to a node and use those for contraints eg the following limits tasks for the redis service to nodes where the node type label equals "queue":

  docker service create \
   --name redis_2 \
   --constraint 'node.labels.type == queue' \
   redis:3.0.6

We can add or remove contraints to a running service - --constraint-add	will add or update a placement constraint and --constraint-rm will remove a constraint
 
We can also specify service placement preferences (--placement-pref) to divide tasks evenly over different categories of nodeseg to balance tasks over a set of datacenters or availability zones eg:
  docker service create \
    --replicas 9 \
    --name redis_2 \
    --placement-pref 'spread=node.labels.datacenter' \
    redis:3.0.6
  Uses the placement-pref "spread" strategy to spread tasks evenly over the values of the datacenter node label.
  
  Multiple placement-prefs can be definwd and the order specified sets the precedence eg:
    docker service create \
      --replicas 9 \
      --name redis_2 \
      --placement-pref 'spread=node.labels.datacenter' \
      --placement-pref 'spread=node.labels.rack' \
      redis:3.0.6
  will spread task evenly across datacentres and then evemly across racks within the datacentres
  
  We can update a service placement-pref using docker service update, --placement-pref-add appends a new placement preference after all existing placement preferences. --placement-pref-rm removes an existing placement preference that matches the argument.
  
