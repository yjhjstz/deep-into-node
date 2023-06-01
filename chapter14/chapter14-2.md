## Node.js Docker

### docker vs host
```js
var http = require('http');
    http.createServer(function (req, res) {
    res.writeHead(200, {'Content-Type': 'text/plain'});
    res.end('Hello World\n');
}).listen(1337); 
console.log('Server running at http://0.0.0.0:1337/');
```
For the performance issue that we concern most, we did a comparative testing on `docker` vs `host` node based on `node-v4.2.3`. The conclusion is that the performance loss is between 1%~4%, depending on network and business code factors. The data is as follows:
- External network environment

```shell
docker node
root@ubuntu-512mb-nyc3-01:~/wrk# ./wrk http://192.241.209.*:1337
Running 10s test @ http://192.241.209.*:1337
  2 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    76.01ms    1.24ms  88.79ms   83.68%
    Req/Sec    65.51     16.12   101.00     76.77%
  1305 requests in 10.04s, 198.81KB read
Requests/sec:    129.99
Transfer/sec:     19.80KB
host node
root@ubuntu-512mb-nyc3-01:~/wrk# ./wrk http://192.241.209.*:1337 
Running 10s test @ http://192.241.209.*:1337
  2 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    75.85ms    1.87ms  87.10ms   61.29%
    Req/Sec    65.66     12.35   101.00     57.07%
  1307 requests in 10.03s, 199.11KB read
Requests/sec:    130.27
Transfer/sec:     19.85KB  
```
- Internal network environment

```shell
** host node**
[root@centos7-x64 statusbar]# wrk  http://localhost:1337
Running 10s test @ http://localhost:1337
  2 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.51ms    1.34ms  39.81ms   98.03%
    Req/Sec     3.46k   821.79     4.29k    76.50%
  68971 requests in 10.05s, 10.26MB read
Requests/sec:   6862.84
Transfer/sec:      1.02MB 

docker node `--net=host` mode
[root@centos7-x64 ~]# wrk http://localhost:1337
Running 10s test @ http://127.0.0.1:1337
  2 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.49ms  287.14us   3.81ms   92.99%
    Req/Sec     3.30k   424.64     3.60k    91.00%
  65866 requests in 10.03s, 9.80MB read
Requests/sec:   6564.82
Transfer/sec:      0.98MB

Overall, for regular web applications, the performance loss is relatively small when connecting to external networks, but it greatly facilitates our development and operation.

### Deploying Applications
> What is MEAN architecture?
> MEAN stands for Mongodb / ExpressJS / AngularJS /NodeJS, which is a combination of popular website application development frameworks that covers frontend to backend. Since these frameworks uses Javascript as the language, it is also commonly referred to as Javascript Fullstack.

In this example, we will attempt to deploy a NodeJS application with MEAN architecture.

```shell
Directory Structure
├── docker-node-full/
│   ├── start.sh
│   ├── Dockerfile
```

### Installing Docker
For Ubuntu system, use apt to install:
```sh
$ sudo apt-get install -y docker-engine
```


### Creation steps
* Create folder
```shell
mkdir ~/docker-node-full && cd $_
```

* Create Dockerfile configuration file

```shell
# Set the base image
FROM ubuntu:14.10

# Install NodeJS and npm
RUN apt-get install -y nodejs npm

# Because the node under apt-get is actually nodejs, create a shortcut for node
RUN ln -s /usr/bin/nodejs /usr/bin/node

# Install Git
RUN apt-get install -y git

# Install Mongodb (from official tutorial)
RUN apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 7F0CEB10
RUN echo 'deb http://downloads-distro.mongodb.org/repo/ubuntu-upstart dist 10gen' | tee /etc/apt/sources.list.d/mongodb.list
RUN apt-get update
RUN apt-get install -y mongodb-org

# Set working directory
WORKDIR /srv/full

# Clean up existing files (if any)
RUN rm -rf /srv/full

# Download the MEAN architecture website code prepared through Git
RUN git clone https://github.com/chuyik/fullstack-demo-dist.git .

# Install NodeJS dependencies
RUN npm install --production

# Create mongodb data folder
RUN mkdir -p /data/db

# Expose ports (for NodeJS application and Mongodb respectively)
EXPOSE 5566 27017

# Set NodeJS application environment variables
ENV NODE_ENV=production PORT=5566

# Add startup script
ADD start.sh /tmp/
RUN chmod +x /tmp/start.sh

# Set default startup command
CMD ["bash", "/tmp/start.sh"]
```

* Create start.sh startup script

```shell
# Start Mongodb in the background
mongod --fork --logpath=/var/log/mongo.log --logappend

# Run NodeJS application
npm start
```

#### Build Image

```shell
# Use this command to build a mirror based on the information configured in Dockerfile
  docker build --rm -t node-full .

# Check if the mirror was created successfully
  docker images
```


#### Run Image

```shell
# Run the image just created
# -p Set port in the format of "host port:container port"
  docker run -p 5566:5566 node-full
```


#### Access the application
You can access http://localhost:5566 with a browser or run curl -s http://localhost:5566.



### Save Mongodb Data Files
Since the Mongodb service runs in the Docker container, the data is also inside, but it is not conducive to data management and preservation. Therefore, we can use some methods to save Mongodb data files outside the container.


#### Disk Mapping
This is the easiest way, and there is a parameter for disk mapping -v in the docker run command.
```shell
# -v Disk mapping, in the format of "host directory:container directory"
docker run -p 5566:5566 -v /var/mongodata:/data/db node-full
```
However, this command fails on Mac and Windows because the boot2docker virtual machine does not support it.
Therefore, you can save the data inside boot2docker and set up a shared folder for Mac or Windows to access.