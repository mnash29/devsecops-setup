# Install Jenkins on AWS EC2
Jenkins is a self-contained Java-based program, ready to run out-of-the-box, with packages for Windows, Mac OS X and other Unix-like operating systems. As an extensible automation server, Jenkins can be used as a simple CI server or turned into the continuous delivery hub for any project.

### Follow this article in **[Youtube](https://youtu.be/-0dkiteJEuE)**

### Prerequisites
- EC2 Amazon Linux 2 x64 t2.micro Instance
	- With Internet Access
	- Security Group with Port `8080` open for internet
- Java v1.8.x 

## Install Java
- Lets install java and set the java home
```sh
yum -y update
yum -y install java-1.8*
```

### Configure Java_Home and Check Version
```sh
vi ~/.bashrc # or whatever file manages user specified values
JAVA_HOME=/usr/lib/jvm/java

export JAVA_HOME

# To set it permanently update your .bash_profile
source ~/.bashrc
java -version
```

_The output should be something like this,_
```
[root@~]# java -version
openjdk version "1.8.0_151"
OpenJDK Runtime Environment (build 1.8.0_151-b12)
OpenJDK 64-Bit Server VM (build 25.151-b12, mixed mode)
```

## Install Jenkins
- You can install jenkins using the rpm or by setting up the repo. We will setup the repo so that we can update it easily in future.
- Get latest version of jenkins from https://pkg.jenkins.io/redhat-stable/
```sh
yum -y install wget
wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
yum -y install jenkins
```

### Start Jenkins
```sh
# Start jenkins service
sudo service jenkins start
```

#### Accessing Jenkins
- By default jenkins runs at port `8080`, You can access jenkins at
```sh
http://YOUR-AWS-IPv4-PUBLIC-IP:8080
```
#### Configure Jenkins
- The default Username is `admin`
- Grab the default password 
  - Password Location:`/var/lib/jenkins/secrets/initialAdminPassword`
- Change admin password
  - `Admin` > `Configure` > `Password`

### Test Jenkins Jobs (optional)
1. Create “new item”
2. Enter an item name – `My-First-Project`
   - Chose `Freestyle` project
3. Under Build section
	Execute shell : echo "Welcome to Jenkins Demo"
4. Save your job 
5. Build job
6. Check "console output"

# Security Considerations AWS Whitepaper

At a minimum, incoming traffic to the Jenkins master should be locked down to the specific IP address
ranges from which you expect traffic. Additionally, your environment can be further secured using the
following methods.

## Enable SSL

Secure Sockets Layer/Transport Layer Security (SSL/TLS) certificates can be used to secure network
communications and establish the identity of websites over the Internet. You can make this easy by fronting your Jenkins master with the ELB load balancer. In that case, you can use the `AWS Certificate Manager` to easily provision a certificate that gives you encrypted network connections and protects your
data as it travels across the wire.

## CSRF Protection

Cross-site request forgery (CSRF) is a class of attack that forces an end user to execute unwanted actions
on Jenkins. By default, Jenkins 2.x installations have the CSRF protection option enabled. To check the
status of this setting, choose `Manage Jenkins`, then `Configure Global Security`, and ensure that Prevent
Cross-Site Request Forgery Exploits is enabled.

## Security Implication of Building on Master

Care should be taken not to perform builds on the master. As you see in the following configuration
steps, we recommend that you configure the master to have no executors, and that you run builds
only on worker nodes. Builds that are run on the master have the ability to read and modify files in
$JENKINS_HOME, which, if accessed maliciously, can affect the entire Jenkins installation and the
security of your system.

## Worker Node Access Control

Jenkins versions 1.5.80.x and later have a subsystem in place that establishes a barrier between worker
nodes and a master to safely allow less trusted worker nodes to be connected to a master.

Your Jenkins 2.x installation should have this barrier enabled by default. You can verify this by visiting
“Manage Jenkins,” then “Configure Global Security,” and ensuring that “Enable Slave->Master Access
Control” is enabled

## Configure User Authentication

During the installation process, you can choose to create a First Admin User. Create a master user that
can be used to create other groups and users, and then proceed to the Jenkins dashboard.

User authentication can be provided through a number of methods:
- The simplest authentication scheme is to use the Jenkins user database. User accounts can be created
using the Jenkins dashboard (choose `Manage Jenkins`, then `Configure Global Security`).
	- The option for Matrix-based security offers the most precise control over user privileges. Using this
option you can specify fine-grained permissions for each user and each action they can take.
- If Jenkins is running on a Windows machine, you can configure Jenkins to authenticate the user name
and the password through Active Directory using the `Active Directory plugin`.
- If Jenkins is running on Linux, the same Active Directory plugin can be used by specifying the Active
Directory domains to authenticate with, or you can configure access to Unix users/groups in your
local system. With this setting, users will be logged into Jenkins by entering their Unix username and
password.
- With the `LDAP plugin`, users can authenticate with an Active Directory-compliant LDAP service such as
`AWS Directory Service` or `OpenLDAP`.

# Worker Node

### Prerequisites
- EC2 Amazon Linux 2 x64 t2.micro Instance
	- With Internet Access
	- Security Group with Port `8080` open for internet
- Java v1.8.x 

### Connect to master instance via ssh

1. Create a pair of authentication keys from the ~/.ssh directory
```sh
cd ~/.ssh
ssh-keygen -t rsa
# accept all defaults
```
2. Copy the public key
```sh
cat ~/.ssh/id_rsa.pub
```
3. From another terminal window, connect to the worker instance via SSH.
4. Edit ~/.ssh/authorized keys and add the contents of the public key from step 5 to the end of the file. Take care not to include any line breaks that you may have copied as you captured the public key.
5. Return to the terminal where you connected to the master instance via SSH, and verify that you can connect to the worker instance the same way.
```sh
ssh ec2-user@YOUR-AWS-IPv4-PUBLIC-IP
```

## Configuration

On the master instance, log in to the Jenkins management dashboard, choose `Manage Jenkins`, then `Manage Nodes`, and finally `New Node`. For each worker node, perform the following steps:
1. Name your node and choose `Permanent Agent`. Then choose `OK` to continue. (If you have already configured a worker node, you can choose `Copy Existing Node` to duplicate a previous configuration.)
2. Add a description and set the number of executors you wish to run on the worker node. Plan for two executors per core.
3. Set `Remote Root Directory` to an absolute path, such as /home/ec2-user/jenkins. The remote root directory will cache data such as tool installations or build workspaces. This prevents unnecessary downloading of tools, or checking out source code again when builds start to run on this agent again after a reboot. Set any other options you want.
4. For the launch method, choose `Launch slave agents on Unix machines via SSH` and set the host to the private IP address of your worker node.
5. Add your credentials by choosing `Add`, then `SSH Username with private key`.
6. Add the value “ec2-user” for `Username`.
7. For Private Key, choose `Enter directly`, then paste in the value of the private key (~/.ssh/id_rsa) from the master instance, and choose `Add`. Finally, choose `Save` to save your worker node configuration.
8. Select the worker from the nodes table, and then choose `Launch agent`.

After you have configured your worker nodes, it’s important to change the number of executors to 0 on the master. Select the master from the nodes table, and configure the “# of executors” settings to 0. This ensures that project builds will now happen on the worker nodes, and not on the master.
Test your Jenkins setup by adding a new project and performing a build.

#  Install & configure Maven build tool on Jenkins
Maven is a code build tool which used to convert your code to artifact. this is widely used plugin to build in continuous integration

#### Prerequisites
- Jenkins worker EC2 Amazon Linux 2 x64 t2.micro Instance
	- With Internet Access
	- Security Group with Port `8080` open for internet
- Java v1.8.x 

#### Install Maven on Jenkins
Download maven packages https://maven.apache.org/download.cgi onto Jenkins server. In this case I am using /opt/maven as my installation directory
- Link : https://maven.apache.org/download.cgi
```sh
  # Creating maven directory under /opt
  mkdir /opt/maven
  cd /opt/maven
  # downloading maven version 3.6.0
  wget http://mirrors.fibergrid.in/apache/maven/maven-3/3.6.0/binaries/apache-maven-3.6.0-bin.zip
  unzip /opt/maven/apache-maven-3.6.0-bin.zip
 ```
Setup M2_HOME and M2 paths in .bash_profile of user and add these to path variable
```sh
  vi ~/.bash_profile
  M2_HOME=/opt/maven/apache-maven-3.6.0
  M2=$M2_HOME/bin
  PAHT=<Existing_PATH>:$M2_HOME:$M2
```
#### Check point 
logoff and login to check maven version
Check maven version 
```sh
  mvn –version
```
So far you have completed installation of maven software to support maven plugin on jenkins console. Let's jump onto jenkins to complete remining steps. 
#### Setup maven on jenkins console
- Install maven plugin without restart  
  - `Manage Jenkins` > `Jenkins Plugins` > `available` > `Maven Invoker`

- Configure java path
  - `Manage Jenkins` > `Global Tool Configuration` > `Maven`