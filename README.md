
wordpress-compose
======================
Deploy a Wordpress + MySQL setup with persistent data using [rexray](https://github.com/emccode/rexray) with a single command

## Description
This docker-compose file will bring up a Wordpress + MySQL pair on a single host. Utilizing [rexray](https://github.com/emccode/rexray) we can persist that data across multiple hosts that is compatible on multiple storage platforms. (Future version will use Docker Machine for easier deployment & libNetwork to have the containers spread across hosts)

## Requirements
* [rexray](https://github.com/emccode/rexray) (as of v0.1.150807) requires CentOS7. This was completed using [AWS ami-96a818fe](http://thecloudmarket.com/image/ami-96a818fe--centos-7-x86-64-2014-09-29-ebs-hvm-b7ee8a69-ee97-4a49-9e68-afaee216db2e-ami-d2a117ba-2)
* [Docker Compose](https://docs.docker.com/compose/) installed on the CentOS 7 hosts. Use `1.4.0` or later [Docker Compose Releases](https://github.com/docker/compose/releases).
* [Docker Engine](https://docs.docker.com/installation/centos/) v 1.8+ installed on the CentOS 7 hosts.

## Installation
Provision 2+ CentOS 7 hosts and issue the following commands on each:
```
$ sudo su -
# yum update -y
# yum install numactl libaio wget -y
# curl -sSL https://get.docker.com/ | sh
# service docker start
# chkconfig docker on
# wget -nv https://github.com/emccode/rexraycli/releases/download/latest/rexray-Linux-x86_64 -O /bin/rexray
# chmod +x /bin/rexray
# wget -nv https://raw.githubusercontent.com/emccode/rexray/master/rexray/rexray.service -O /usr/lib/systemd/system/rexray.service

//change your Environment Variables depending upon your storage system. Consult the rexray documenation to learn more
# echo 'AWS_SECRET_KEY=mysecretkey' >> /etc/environment
# echo 'AWS_ACCESS_KEY=myaccesskey' >> /etc/environment

# systemctl enable rexray.service
# systemctl start rexray.service
# exit
$ sudo su -
```

At this point you should be able to do a `docker info` to see Docker information and `rexray get-instance` that shows what storage platforms are available to rexray.
```
# docker info
Containers: 0
Images: 0
Storage Driver: devicemapper
 Pool Name: docker-202:1-8589003-pool
 Pool Blocksize: 65.54 kB
 Backing Filesystem: xfs
 Data file: /dev/loop0
 Metadata file: /dev/loop1
 Data Space Used: 1.821 GB
 Data Space Total: 107.4 GB
 Data Space Available: 7.611 GB
 Metadata Space Used: 1.479 MB
 Metadata Space Total: 2.147 GB
 Metadata Space Available: 2.146 GB
 Udev Sync Supported: true
 Deferred Removal Enabled: false
 Data loop file: /var/lib/docker/devicemapper/devicemapper/data
 Metadata loop file: /var/lib/docker/devicemapper/devicemapper/metadata
 Library Version: 1.02.93-RHEL7 (2015-01-28)
Execution Driver: native-0.2
Logging Driver: json-file
Kernel Version: 3.10.0-123.8.1.el7.x86_64
Operating System: CentOS Linux 7 (Core)
CPUs: 1
Total Memory: 992.8 MiB
Name: ip-172-31-24-237
ID: XZZY:OI2U:VW4X:L46Q:WJP2:FIIS:O7NI:HHSV:J5IC:RLQV:JHNO:ZE56
# rexray get-instance
- providername: ec2
  instanceid: i-e7f57c35
  region: us-east-1
  name: ""
```

Install the latest version of [docker-compose](https://github.com/docker/compose/releases) (must be 1.4+) on each CentOS 7 machine
```
# curl -L https://github.com/docker/compose/releases/download/1.4.0/docker-compose-`uname -s`-`uname -m` > /bin/docker-compose
# chmod +x /bin/docker-compose
```

## Provision Volumes for Persistent Data
The data that will persist between containers that are destroyed and created must live on the storage platform. Using rexray, we will create two new volumes to store MySQL and Wordpress data. Each of these volumes is 20GB in size. You can size them however you want and depending on the platform (EBS in this case) you can specify the backing type and IOPS needed. The most important variable is the name given to the volume because that must be passed to the `docker run...` command. Only run this command on 1 host
```
# rexray new-volume --size=20 --volumename="wpdata"
# rexray new-volume --size=20 --iops=300 --volumetype="ssd" --volumename="mysqldata"
```

From a different host you can run `rexray get-volume` to see the new volume is accessible between hosts.

## Start Containers with Docker Compose
We will grab this `docker-compose.yml` file to bring up the containers on the same host. Only run this on a single host.
```
# mkdir /home/wordpress-compose
# curl -L https://raw.githubusercontent.com/kacole2/wordpress-compose/master/docker-compose.yml > /home/wordpress-compose/docker-compose.yml
# cd /home/wordpress-compose
# docker-compose up -d
```

## Manipulate Some Data!
You should now be able to access the Wordpress Installation screen by accessing port 8080 on the host. Complete the Wordpress Installation.
![Complete the Wordpress Installation](https://s3.amazonaws.com/kennyonetime/wordpress-compose01.png)

Login to the Wordpress Admin side, Create A New Post and Click on Publish
![Click on Publish](https://s3.amazonaws.com/kennyonetime/wordpress-compose03.png)

Visit the site and you should now see a new blog post. We now have data!
![Visit the site](https://s3.amazonaws.com/kennyonetime/wordpress-compose04.png)

## Kill the containers and restart them somewhere else
We are going to kill the containers and restart them on a different host
```
# docker-compose kill
```

On a different host we will perform the same steps as before
```
# mkdir /home/wordpress-compose
# curl -L https://raw.githubusercontent.com/kacole2/wordpress-compose/master/docker-compose.yml > /home/wordpress-compose/docker-compose.yml
# cd /home/wordpress-compose
# docker-compose up -d
```

Now you should be able to access your Wordpress site from a different IP address with all the data still intact. We are achieving data persistence!
![Visit the site](https://s3.amazonaws.com/kennyonetime/wordpress-compose04.png)

### Start Containers Manually
If you don't want to use Docker Compose, you can start the containers manually with:
```
# docker run -d --name wpmysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD="87654321" --volume-driver=rexray -v mysqldata:/var/lib/mysql mysql:latest

# docker run --name wordpress --link wpmysql:mysql -e WORDPRESS_DB_PASSWORD="87654321" -p 8080:80 --volume-driver=rexray -v wpdata:/var/www/html -d wordpress
```

## Contribution
Create a fork of the project into your own reposity. Make all your necessary changes and create a pull request with a description on what was added or removed and details explaining the changes in lines of code. If approved, project owners will merge it.

## Licensing
wordpress-compose is freely distributed under the [MIT License](http://opensource.org/licenses/MIT). See LICENSE for details.

## Support
Please file bugs and issues at the Github issues page.