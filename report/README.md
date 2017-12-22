title: Lab 04 - Docker
---

## Lab 04 - Docker


### Table of contents
1. [Introduction](#introduction)
2. [Task 0: Identify issues and install the tools](#paragraph1)
3. [Task 1: Add a process supervisor to run several processes](#paragraph2)
4. [Task 2: Add a tool to manage membership in the web server cluster](#paragraph3)
5. [Task 3: React to membership changes](#paragraph4)
6. [Task 4: Use a template engine to easily generate configuration files](#paragraph5)
7. [Task 5: Generate a new load balancer configuration when membership changes](#paragraph6)
8. [Task 6: Make the load balancer automatically reload the new configuration](#paragraph7)
9. [Difficulties](#difficulties)
10. [Conclusion](#conclusion)



### Introduction <a name="introduction"></a>

The aim of this lab is to build our own Docker image. Through these steps, we'll become 
familiar with the process supervision for Docker, understand the concept for dynamic scaling 
of an app and, finally, put into practice decentralized management of web server instances. 
The lab is divided in seven parts. The version of HAProxy used is 1.5. 


### <a name="paragraph1"></a>Task 0: Identify issues and install the tools 

#### Identify issues

Suppose further currently your web servers and your load balancer are
deployed like in the previous lab. What are the issues with this
architecture? 

1. <a name="M1"></a>**[M1]** __Do you think we can use the current
   solution for a production environment? What are the main problems
   when deploying it in a production environment?__
   
   <a name="M1"></a>**[A1]** No, it's not a good idead to user it in
   a production environment. The main problpem is that we have to 
   delacre each server manually in the conf file (it's a static 
   configuration).It is not made automatically, so it's a lof of work 
   to maintain. Beside, each time we add a server, we need to reboot. 

2. <a name="M2"></a>**[M2]** __Describe what you need to do to add new
   `webapp` container to the infrastructure. Give the exact steps of
   what you have to do without modifiying the way the things are
   done. Hint: You probably have to modify some configuration and
   script files in a Docker image.__

   <a name="M1"></a>**[A2]** We need to change ha/config/haproxy.cfg file
   and add a new line : 
   ```bash
      server s3 <s3>:3000 check
   ```
   Same thing in the ha/scripts/run.sh file:
   ```bash
   sed -i 's/<s3>/$S3_PORT_3000_TCP_ADDR/g' /usr/local/etc/haproxy/haproxy.cfg
   ```
   Since we modify the config of ha container, we need to re-build it. When 
   it's done, we can normally run the ha container and don't forget to run also
   the new server s3.

3. <a name="M3"></a>**[M3]** __Based on your previous answers, you have
   detected some issues in the current solution. Now propose a better
   approach at a high level.__

   <a name="M1"></a>**[A3]** The configuration should not be static but dynamic.
   We should use a tool or a program to communicate with te load balancer to tell
   it which server are up or down. 

4. <a name="M4"></a>**[M4]** __You probably noticed that the list of web
  application nodes is hardcoded in the load balancer
  configuration. How can we manage the web app nodes in a more dynamic
  fashion?__

  <a name="M1"></a>**[A4]** As said previously, we should user a special tool that 
  will say to the load balancer all servers that are connected. For this lab, we 
  will be introduced to the serf agent. 

5. <a name="M5"></a>**[M5]** __In the physical or virtual machines of a
   typical infrastructure we tend to have not only one main process
   (like the web server or the load balancer) running, but a few
   additional processes on the side to perform management tasks.
   For example to monitor the distributed system as a whole it is
   common to collect in one centralized place all the logs produced by
   the different machines. Therefore we need a process running on each
   machine that will forward the logs to the central place. (We could
   also imagine a central tool that reaches out to each machine to
   gather the logs. That's a push vs. pull problem.) It is quite
   common to see a push mechanism used for this kind of task.
   Do you think our current solution is able to run additional
   management processes beside the main web server / load balancer
   process in a container? If no, what is missing / required to reach
   the goal? If yes, how to proceed to run for example a log
   forwarding process?__

   <a name="M1"></a>**[A5]** We need a central tool that will bypass our 
   problem of one process per container. In this lab, we will use s6. 

6. <a name="M6"></a>**[M6]** __In our current solution, although the
   load balancer configuration is changing dynamically, it doesn't
   follow dynamically the configuration of our distributed system when
   web servers are added or removed. If we take a closer look at the
   `run.sh` script, we see two calls to `sed` which will replace two
   lines in the `haproxy.cfg` configuration file just before we start
   `haproxy`. You clearly see that the configuration file has two
   lines and the script will replace these two lines.
   What happens if we add more web server nodes? Do you think it is
   really dynamic? It's far away from being a dynamic
   configuration. Can you propose a solution to solve this?__

   <a name="M1"></a>**[A6]** No, it's not, because we need to change two files to 
   add more nodes (see answer 2). A better solution would be that each server announces
   itself to the other and the load balancer when it's up. 

#### Install the tools

We already set up everything in the previous lab. Here is our output after the following command : 
```bash
vagrant@ubuntu-14:~$ docker ps
CONTAINER ID        IMAGE                  COMMAND                  CREATED             STATUS              PORTS                                                                NAMES
ca41d450c676        softengheigvd/ha       "/docker-entrypoint.…"   24 seconds ago      Up 23 seconds       0.0.0.0:80->80/tcp, 0.0.0.0:1936->1936/tcp, 0.0.0.0:9999->9999/tcp   ha
c160fa8469fa        softengheigvd/webapp   "/scripts/run.sh"        24 seconds ago      Up 23 seconds       3000/tcp                                                             s2
78a6d9e876a0        softengheigvd/webapp   "/scripts/run.sh"        25 seconds ago      Up 24 seconds       3000/tcp                                                             s1
```


**Deliverables**:

1. __Take a screenshot of the stats page of HAProxy at
   <http://192.168.42.42:1936>. You should see your backend nodes.__

   ![alt img](./img/task0_answer1.PNG)

2. __Give the URL of your repository URL in the lab report.__

https://github.com/LuanaMartelli/Teaching-HEIGVD-AIT-2016-Labo-Docker


### <a name="paragraph2"></a>Task 1: Add a process supervisor to run several processes

> In this task, we will learn to install a process supervisor that
  will help us to solve the issue presented in the question
  [M5](#M5). Installing a process supervisor gives us the ability to
  run multiple processes at the same time in a Docker environment.


**Deliverables**:

1. __Take a screenshot of the stats page of HAProxy at
   <http://192.168.42.42:1936>. You should see your backend nodes. It
   should be really similar to the screenshot of the previous task.__

   ![alt img](./img/task1_ha_statistics.png)

2. __Describe your difficulties for this task and your understanding of
   what is happening during this task. Explain in your own words why
   are we installing a process supervisor. Do not hesitate to do more
   research and to find more articles on that topic to illustrate the
   problem.__

   As said in the [A5](#A5), we cannot run more than one process per 
   container. Thanks to a supervisor, we ca bypass this limitation. 
   Now, we will be able to run a server and a process to log it. 
   The main problem for this part was to understand all the theroy 
   behind the idea of a supervisor process and how Docker / virtual machines
   run.
   

### <a name="paragraph3"></a>Task 2: Add a tool to manage membership in the web server cluster

In this task, we will focus on how to make our infrastructure more
flexible so that we can dynamically add and remove web servers. To
achieve this goal, we will use a tool that allows each node to know
which other nodes exist at any given time.


**Deliverables**:

1. __Provide the docker log output for each of the containers: `ha`,
   `s1` and `s2`.__

   We run the HAProxy first. The logs are avaliables under /logs/task2/Ha_Start_First/


2. __Give the answer to the question about the existing problem with the
   current solution.__

   First run haproxy. If you do not, the child nodes will not be linked to it


3. __Give an explanation on how `Serf` is working. Read the official
   website to get more details about the `GOSSIP` protocol used in
   `Serf`. Try to find other solutions that can be used to solve
   similar situations where we need some auto-discovery mechanism.__

   TODO



### <a name="paragraph4"></a>Task 3: React to membership changes


We reached a state where we have nearly all the pieces in place to make the infrastructure
really dynamic. At the moment, we are missing the scripts that will react to the events
reported by `Serf`, namely member `leave` or member `join`.

We will start by creating the scripts in [ha/scripts](ha/scripts). So create two files in
this directory and set them as executable. 


**Deliverables**:

1. __Provide the docker log output for each of the containers:  `ha`, `s1` and `s2`.
   Put your logs in the `logs` directory you created in the previous task.__

   The logs are available in the files   
   	- /logs/task3/ha.txt
   	- /logs/task3/s1.txt
   	- /logs/task3/s2.txt


3. __Provide the logs from the `ha` container gathered directly from the `/var/log/serf.log`
   file present in the container. Put the logs in the `logs` directory in your repo.__

   Here are the logs we got :
```
	Member join script triggered
	Member join event received from: 2a70b5b171fb with role balancer
	Member join script triggered
	Member join event received from: ade84b5d0a98 with role backend
	Member join script triggered
	Member join event received from: 2fd4ce8bff6a with role backend
```
   They are available in the files   
   	- /logs/task3/serf_logs/serf.txt


### <a name="paragraph5"></a>Task 4: Use a template engine to easily generate configuration files

There are several ways to generate a configuration file from variables
in a dynamic fashion. In this lab we decided to use `NodeJS` and
`Handlebars` for the template engine.  
In our case our template is the `HAProxy` configuration file in which
we put placeholders written in the template language. Our data model
is the data provided by the handler scripts of `Serf`. And the
resulting document coming out of the template engine is a
configuration file that HA proxy can understand where the placeholders
have been replaced with the data.


**Deliverables**:

1. __You probably noticed when we added `xz-utils`, we have to rebuild
   the whole image which took some time. What can we do to mitigate
   that? Take a look at the Docker documentation on
   [image layers](https://docs.docker.com/engine/userguide/storagedriver/imagesandcontainers/#images-and-layers).
   Tell us about the pros and cons to merge as much as possible of the
   command. In other words, compare:__

  ```
  RUN command 1
  RUN command 2
  RUN command 3
  ```

  __vs.__

  ```
  RUN command 1 && command 2 && command 3
  ```

  A Docker image is built up from layers. Each layer is an instruction in the Dockerfile. It is a better practice to have a minimal number of layers in an image, because the size can become fat really quickly. That's why put all commands in one RUN is a better practice. It will reduce the size of the image. On an other hand, each layer is cached, so when an image is re-built, it could go faster to not re-download all the packages. So in this case, the first solution is better. 
  An other problem with a merge of commands, like presented here, is that it becomes quickly not really easy to read when there is a lot of instructions.  
  Regarding the apt-get update and apt-get install, there are a lot of articles, saying that they should be put in one RUN command, because of a cache reason. A single apt-get update will be cached, as said previously, and not re-run every time you need to install something. So it might download an old version of the package. 


  There are also some articles about techniques to reduce the image
  size. Try to find them. They are talking about `squashing` or
  `flattening` images.

2. __Propose a different approach to architecture our images to be able
   to reuse as much as possible what we have done. Your proposition
   should also try to avoid as much as possible repetitions between
   your images.__

3. __Provide the `/tmp/haproxy.cfg` file generated in the `ha` container
   after each step.  Place the output into the `logs` folder like you
   already did for the Docker logs in the previous tasks. Three files
   are expected.__
   
   In addition, provide a log file containing the output of the 
   `docker ps` console and another file (per container) with
   `docker inspect <container>`. Four files are expected.
   
4. __Based on the three output files you have collected, what can you
   say about the way we generate it? What is the problem if any?__



### <a name="paragraph6"></a>Task 5: Generate a new load balancer configuration when membership changes

At this stage, we have:

  - Two images with `S6` process supervisor that starts a Serf agent
    and an "application" (HAProxy or Node web app).

  - The `ha` image contains the required stuff to react to `Serf`
    events when a container joins or leaves the `Serf` cluster.

  - A template engine in the `ha` image is ready to be used to
    generate the HAProxy configuration file.

Now, we need to refine our `join` and `leave` scripts to generate a
proper HAProxy configuration file.


**Deliverables**:

1. __Provide the file `/usr/local/etc/haproxy/haproxy.cfg` generated in
   the `ha` container after each step. Three files are expected.
   In addition, provide a log file containing the output of the 
   `docker ps` console and another file (per container) with
   `docker inspect <container>`. Four files are expected.__

2. __Provide the list of files from the `/nodes` folder inside the `ha` container.
   One file expected with the command output.__

3. __Provide the configuration file after you stopped one container and
   the list of nodes present in the `/nodes` folder. One file expected
   with the command output. Two files are expected.
   In addition, provide a log file containing the output of the
   `docker ps` console. One file expected.__

4. __(Optional:) Propose a different approach to manage the list of backend
   nodes. You do not need to implement it. You can also propose your
   own tools or the ones you discovered online. In that case, do not
   forget to cite your references.__



### <a name="paragraph7"></a>Task 6: Make the load balancer automatically reload the new configuration

The only thing missing now is to make sure the configuration of
HAProxy is up-to-date and taken into account by HAProxy.

We will try to make HAProxy reload his config with minimal
downtime. At the moment, we will replace the line `TODO: [CFG] Replace
this command` in [ha/services/ha/run](ha/services/ha/run) by the
following script part. As usual, take the time to read the comments.


**Deliverables**:

1. __Take a screenshots of the HAProxy stat page showing more than 2 web
   applications running. Additional screenshots are welcome to see a
   sequence of experimentations like shutting down a node and starting
   more nodes.
   Also provide the output of `docker ps` in a log file. At least 
   one file is expected. You can provide one output per step of your
   experimentation according to your screenshots.__
   
2. __Give your own feelings about the final solution. Propose
   improvements or ways to do the things differently. If any, provide
   references to your readings for the improvements.__

3. __(Optional:) Present a live demo where you add and remove a backend container.__



## <a name="difficulties"></a> Difficulties

Windows.



## <a name="conclusion"></a> Conclusion

That was nice 