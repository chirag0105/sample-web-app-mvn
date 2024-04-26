# Setting up EC2 instances
1. Register for [AWS](https://aws.amazon.com/).
2. Once registered go to the EC2 panel and make sure you are in the correct region (in the top left).

<img width="1392" alt="ec2" src="https://github.com/chirag0105/sample-web-app-mvn/assets/120470330/bec2f653-1859-46f1-bc4c-566cf1bd7459">

3. Once your on the EC2 panel, Click `Instances (Running)` and then start a new instance with the following settings:
  - Amazon Linux 2023 AMI
  - t2.micro (to avoid charge)
  - Make a new keypair (call it whatever you choose)
  - Name the machine `jenkins`

4. Make another one of these instances but do the following:
  - Use the same keypair as `jenkins`
  - Name the machine `apache`

5. Once built, go back to the `EC2 Dashboard` and select the `jenkins` machine then press `Connect`, then scroll down in the new tab and press `Connect` again.

6. You are now setup on EC2

# Installing Jenkins on your `jenkins` instance

We will be following [these steps](https://www.jenkins.io/doc/tutorials/tutorial-for-installing-jenkins-on-AWS/)

1. In your connected terminal run the following commands (in order, waiting for each to complete):
```
sudo yum update –y
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
sudo dnf install java-17-amazon-corretto -y
sudo yum install jenkins -y
sudo systemctl enable jenkins
sudo systemctl start jenkins
```

2. Once all executed check the status with `sudo systemctl status jenkins` and it should be `active` (and preferably green!). Leave with 'q'.

3. If you try to visit your instances public IP on port 8080 using http (http://MY.IP.HERE:8080) you won't get a response as your security groups are wrong!

## Fixing the security groups

1. Go to your `jenkins` instance scroll down and select the `Security` tab, then click on the blue hyperlink to your security group under the Security Groups header, it should have something like `(launch-wizard-x)` towards the end.

2. Press `Actions` then `Edit inbound rules`.

3. Add a rule with the following settings:
- Type: Custom TCP
- Port Range: 8080
- Source: Anywhere-IPV4

4. Save your changes.

5. Do the same for the `apache` instance.

6. You should now be able to connect to Jenkins.

## Jenkins Config

1. You will be presented with an 'Unlock Jenkins' page, if all goes well.

2. The password can be found by putting `sudo cat /var/lib/jenkins/secrets/initialAdminPassword` into the console, the output is your password to unlock.

3. Make your admin user, give it a password you will remember and assign an email.

4. It will ask about plugins, install the suggested ones.

5. Once you are in you will be presented with the Jenkins dashboard. Nice job!

## Installing Apache Tomcat onto `apache`

We will be following steps [from here](https://medium.com/@raguyazhin/step-by-step-guide-to-install-apache-tomcat-on-amazon-linux-120748a151a9) but with some changes.

1. Make sure to `Connect` to the `apache` instance the same way you did with `jenkins`, just make sure you are on the correct one.

2. Run the following commands (in order):
   
```
yum install java-1.8*
sudo su -
cd /opt
wget https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.88/bin/apache-tomcat-9.0.88.tar.gz
tar -zvxf apache-tomcat-9.0.88.tar.gz
cd apache-tomcat-9.0.88
cd bin
chmod +x startup.sh
chmod +x shutdown.sh
./startup.sh
cd ..
```

3. Now we are going to edit each of the following files output from this command: `find -name context.xml`.

```
find -name context.xml
./conf/context.xml
./webapps/examples/META-INF/context.xml
./webapps/host-manager/META-INF/context.xml
./webapps/manager/META-INF/context.xml
```

4. You need to edit each with `nano` like this:

If a file contains `<Valve ...>` make sure to comment out the line by putting a `<!--` before and a `-->` after `/>`.

Eg. `<Valve ... />` to `<!--<Valve ... />-->`

5. Afterwards do `cd conf` then `nano tomcat-users.xml` and append the codeblock below after the line containing `<tomcat-users>`.

```
<role rolename="manager-gui"/>
<role rolename="manager-script"/>
<role rolename="manager-jmx"/>
<role rolename="manager-status"/>   
<user username="admin" password="admin" roles="manager-gui,manager-script,manager-jmx,manager-status"/>
<user username="deployer" password="deployer" roles="manager-script"/>
<user username="tomcat" password="s3cret" roles="manager-gui"/>
```

This creates 3 users:
```
admin:admin
deployer:deployer
tomcat:s3cret
```

6. If you visit `http://APACHE_INSTANCE_IP:8080` you should see an Apache Tomcat instance!

## Using Jenkins to Deploy to Apache Tomcat

### Fixing the Built-in Node

1. Go back to `http://JENKINS_INSTANCE_IP:8080`

2. If you were logged out, log in with your credentials. Go to `Manage Jenkins`, then `Nodes`, then press the cog to the right of `Built-in Node`.

3. Under `Node Properties`, enable `Disk Space Monitoring Thresholds` and set all the values to `100MB` and save.

4. Press `Bring this node back online` (sometimes twice if it remains after the first press.

5. Return to the home page.

### Configuring Maven

1. Go to `Manage Jenkins`, then `Tools`, then scroll down to `Maven installations` and add a new installation.

2. Call it `latest-maven` and make the version the latest (as of writing `3.9.6`).

3. Press `Save` and return to the home page.

### Making a job

1. Press `New Item`.

2. Name it however you feel, and select it as a `Freestyle project`, then press `Ok`.

3. Give a description of some sort. Then scroll down to `Source Code Management` and select `Git`. Set the Repository URL to `https://github.com/chirag0105/sample-web-app-mvn` and under `Branches to build` make sure to specify the `*/main`.

4. Scroll down to `Build Steps` and add a new step `Invoke top-level Maven targets`. Set the following options:

```
Maven Version: latest-maven
Goals: clean package
```

5. Scroll down further to `Post-build Actions` and add a `Deploy war/ear to a container`. Set the following options:

```
WAR/EAR files: target/*
Containers: Tomcat 9.x Remote
```

6. Inside the `Tomcat 9.x Remote` container settings, for the Credentials press the `Add` then `Jenkins` and set the Username to `admin` and the Password to `admin`, then scroll down and press `Add`.

7. Select the `admin/*****` credential and set the Tomcat URL to `http://TOMCAT_IP_ADDRESS:8080`

8. Press `Save` and then press `Build Now`.

9. If you go to `http://TOMCAT_IP_ADDRESS:8080//sample-web-app-mvn/` once the build is complete, you should have a working webapp. Nice job!

## Congrats!
