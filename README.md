**Summary of Tasks:**

1. Create a virtual machine on AWS
2. Setup docker and docker-compose on VM
3. Write a docker-compose file
4. Write a dockerfile
5. Execute docker-compose file
6. Create a database in MySQL container
7. Install MYSQL and AWS CLI in the remote container
8. Create mysql dump from remote container
9. Create S3 bucket and an IAM user
10. Store MYSQL dump into S3
11. Automate this whole process through bash script
12. Automate this whole process through the Jenkins dashboard

---

## Create a virtual machine (EC2)
Create a virtual machine on AWS with the following configuration.

![image 1](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/vicezu814gfaztn33uid.PNG)

![image 2](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/t5aawu6qit8gcu0jlwgx.PNG)

![image 3](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/138swfmavu38wf6xz5es.PNG)

![image 4](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/vryc0bbf9f89da72a69m.PNG)

## Setup docker and docker-compose on VM
```
yum install docker -y
curl -SL https://github.com/docker/compose/releases/download/v2.20.3/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose;
service docker start
```

## Write a docker-compose file

```
version: '3.3'

services:
  jenkins:
    container_name: jenkins
    image: jenkins/jenkins
    ports:
      - "8080:8080"
    volumes:
      - ./jenkins_home:/var/jenkins_home
    networks:
      - jenkins_net

  remote_host:
    container_name: remote_host
    image: remote-host
    build:
      context: ./remote_container
    networks:
      - jenkins_net

  db_host:
    container_name: db
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: 1234
    volumes:
      - ./db_data:/var/lib/mysql
    networks:
      - jenkins_net

networks:
  jenkins_net:

```


## Write a dockerfile

```
FROM centos:7

RUN yum -y install openssh-server passwd
RUN useradd -m remote_user && \
    echo "password" | passwd --stdin remote_user

RUN mkdir -p /home/remote_user/.ssh && \
    chmod 700 /home/remote_user/.ssh

COPY remote-key.pub /home/remote_user/.ssh/authorized_keys

RUN chown remote_user:remote_user -R /home/remote_user/.ssh && \
    chmod 600 /home/remote_user/.ssh/authorized_keys

RUN /usr/bin/ssh-keygen -A

EXPOSE 22

RUN rm -f /run/nologin

CMD ["/usr/sbin/sshd", "-D"]
```

## Execute docker-compose file
first build image using following command

```
docker-compose build 
```
Now create containers with this image

```
docker-compose run -d
```

## Create a database in MySQL container
login the db container with the following command

```
docker exec -it db bash
```
Now run command to interact with MySQL CLI

```
MySQL -u root -p1234
```
Here create a database with the name 'cloud' and insert some information in this database

```
CREATE DATABASE cloud;
SHOW DATABASES;

CREATE TABLE cloud.country (
  `Code` CHAR(3) NOT NULL DEFAULT '',
  `Name` CHAR(52) NOT NULL DEFAULT '',
  `Conitinent` enum('Asia','Europe','North America','Africa','Oceania','Antarctica','South  America') NOT NULL DEFAULT 'Asia',
  `Region` CHAR(26) NOT NULL DEFAULT '',
  `SurfaceArea` FLOAT(10,2) NOT NULL DEFAULT '0.00',
  `IndepYear` SMALLINT(6) DEFAULT NULL,
  `Population` INT(11) NOT NULL DEFAULT '0',
  `LifeExpectancy` FLOAT(3,1) DEFAULT NULL,
  `GNP` FLOAT(10,2) DEFAULT NULL,
  `GNPOld` FLOAT(10,2) DEFAULT NULL,
  `LocalName` CHAR(45) NOT NULL DEFAULT '',
  `GovernmentForm` CHAR(45) NOT NULL DEFAULT '',
  `HeadOfState` CHAR(60) DEFAULT NULL,
  `Capital` INT(11) DEFAULT NULL,
  `Code2` CHAR(2) NOT NULL DEFAULT '',
  PRIMARY KEY (`Code`)
);

USE cloud;
SHOW TABLES;
SHOW COLUMNS FROM cloud.country;
```

## Install MYSQL and AWS CLI in the remote container
Exit from db container and modify dockerfile add additional commands to install MySQL and was cli.

```
RUN yum -y install mysql
RUN yum -y install epel-release && \
    yum -y install python3-pip && \
    pip3 install --upgrade pip && \
    pip3 install awscli
```
Now your dockerfile will look like this

![image 5](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ztr3opwa5zpxuww4y94x.PNG)
After modification, build image and run containers again.

## Create mysql dump from remote container
first login the remote container and create dump

```
docker exec -it remote_host bash
```
Now create dump and store it into tmp directory of remote container

```
mysqldump -h db -u root -p cloud > /tmp.cloud.sql
```
you can check that a dump has been created in tmp directory

## Create S3 bucket and an IAM user
simply create a S3 bucket and an IAM user with following or your own configuration and use default settings.

![myimage](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/y9w4nzy8eouypcfigrri.PNG)

![newimage](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/q6og8u6gsv8zsi7ez3ns.PNG)

Now we will create a user with the following settings.

![I am user1](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/fve70plb8h4tmqm9nozs.PNG)
![Imagyes](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/d5bcdrn7b09cyu4wt35v.PNG)

![Image deshelloon](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/kgzhwzlof6asv05awlll.PNG)

We will user IAM user credentials and paste it in remote container cli

```
export AWS_ACCESS_KEY_ID=AKIASFFF6PNAP7MHNCRV
export AWS_SECRET_ACCESS_KEY=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
export AWS_DEFAULT_REGION=us-east-1
```

## Store MYSQL dump into S3
Now we will run a command to store MySQL dump into AWS S3 storage.

```
aws s3 cp /tmp/cloud.sql s3://cloud-bucket-1
```
after run this command you can check through console that a copy of cloud.sql is uploaded in aws s3

## Automate this whole process through bash script
Now we will write a bash script to automate this whole process. make a file with the name of s3.sh

```
#!/bin/bash

DATE=$(date +%H-%M-%S)
DB_HOST=$1
DB_PASSWORD=$2
DB_NAME=$3
AWS_SECRET=$4
BUCKET_NAME=$5

mysqldump -h $DB_HOST -u root -p$DB_PASSWORD $DB_NAME > /tmp/cloud-$DATE.sql && \
export AWS_ACCESS_KEY_ID=AKIA36LN4QIKPOKDJEUM && \
export AWS_SECRET_ACCESS_KEY=$AWS_SECRET && \
export AWS_DEFAULT_REGION=us-east-1
echo "--- Uploading database dump to aws s3 bucket ---"
aws s3 cp /tmp/cloud-$DATE.sql s3://$BUCKET_NAME/cloud-$DATE.sql
```
Now login the remote container and run the following command .

```
./s3.sh db 1234 cloud 'secret access key of user' 'bucket name'
```
as often as you will run this command in remote container you will get copies of dump so many times.

## Automate this whole process through the Jenkins dashboard
To know how can we automate this process using jerking dashboard you can watch my complete video of this project.
[![IMAGE ALT TEXT HERE](https://img.youtube.com/vi/VOiF8ARr2BM/0.jpg)](https://www.youtube.com/watch?v=VOiF8ARr2BM)














