## Install Nexus Repository Manager
   ### Use the following script in userdata
   ```
      #!/bin/bash
      yum install java-1.8.0-openjdk.x86_64 wget -y   
      mkdir -p /opt/nexus/   
      mkdir -p /tmp/nexus/                           
      cd /tmp/nexus/
      NEXUSURL="https://download.sonatype.com/nexus/3/latest-unix.tar.gz"
      wget $NEXUSURL -O nexus.tar.gz
      EXTOUT=`tar xzvf nexus.tar.gz`
      NEXUSDIR=`echo $EXTOUT | cut -d '/' -f1`
      rm -rf /tmp/nexus/nexus.tar.gz
      rsync -avzh /tmp/nexus/ /opt/nexus/
      useradd nexus
      chown -R nexus.nexus /opt/nexus 
      cat <<EOT>> /etc/systemd/system/nexus.service
      [Unit]                                                                          
      Description=nexus service                                                       
      After=network.target                                                            
                                                                        
      [Service]                                                                       
      Type=forking                                                                    
      LimitNOFILE=65536                                                               
      ExecStart=/opt/nexus/$NEXUSDIR/bin/nexus start                                  
      ExecStop=/opt/nexus/$NEXUSDIR/bin/nexus stop                                    
      User=nexus                                                                      
      Restart=on-abort                                                                
                                                                        
      [Install]                                                                       
      WantedBy=multi-user.target                                                      
      
      EOT
      
      echo 'run_as_user="nexus"' > /opt/nexus/$NEXUSDIR/bin/nexus.rc
      systemctl daemon-reload
      systemctl start nexus
      systemctl enable nexus
   ```

## Install Apache Maven

   ### Follow the steps below to Create Maven Server

      The following are instructions for installing Apache Maven and Java 8 on an Amazon EC2 instance. These are required for the Amazon Neptune Signature Version 4 authentication samples.
      
      ## Steps To Install Apache Maven and Java 8 on your EC2 instance
      
      1. Connect to your Amazon EC2 instance with an SSH client.
      
      2. Install Apache Maven on your EC2 instance. First, enter the following to add a repository with a Maven package.
      
       ``sudo wget https://repos.fedorapeople.org/repos/dchen/apache-maven/epel-apache-maven.repo -O /etc/yum.repos.d/epel-apache-maven.repo``
      
      - Enter the following to set the version number for the packages.
      
       `sudo sed -i s/\$releasever/6/g /etc/yum.repos.d/epel-apache-maven.repo`
      - Then you can use yum to install Maven.
      
       `sudo yum install -y apache-maven`
      3. The Gremlin libraries require Java 8. Enter the following to install Java 8 on your EC2 instance.
      
       `sudo yum install java-1.8.0-devel`
      4. Enter the following to set Java 8 as the default runtime on your EC2 instance.
      
       `sudo /usr/sbin/alternatives --config java`
      - When prompted, enter the number for Java 8.
      
      5. Enter the following to set Java 8 as the default compiler on your EC2 instance.
      
       `sudo /usr/sbin/alternatives --config javac`
      - When prompted, enter the number for Java 8.
      
      6. Verify your maven version
       `mvn -v`
      
      ### Project Preparation
      7. Create the `.m2` directory in the home directory of your current user
       `mkdir ~/.m2`
      
      8. Create the Settings file inside of the `~/.m2` directory
       `cd ~/.m2/`
       `mv demo/settings.xml ~/.m2/`


## Install SonarQube
Use the following script in User-data
```
   #!/bin/bash
   cp /etc/sysctl.conf /root/sysctl.conf_backup
   cat <<EOT> /etc/sysctl.conf
   vm.max_map_count=262144
   fs.file-max=65536
   ulimit -n 65536
   ulimit -u 4096
   EOT
   cp /etc/security/limits.conf /root/sec_limit.conf_backup
   cat <<EOT> /etc/security/limits.conf
   sonarqube   -   nofile   65536
   sonarqube   -   nproc    409
   EOT
   
   sudo apt-get update -y
   sudo apt-get install openjdk-11-jdk -y
   sudo update-alternatives --config java
   
   java -version
   
   sudo apt update
   wget -q https://www.postgresql.org/media/keys/ACCC4CF8.asc -O - | sudo apt-key add -
   
   sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" >> /etc/apt/sources.list.d/pgdg.list'
   sudo apt install postgresql postgresql-contrib -y
   #sudo -u postgres psql -c "SELECT version();"
   sudo systemctl enable postgresql.service
   sudo systemctl start  postgresql.service
   sudo echo "postgres:admin123" | chpasswd
   runuser -l postgres -c "createuser sonar"
   sudo -i -u postgres psql -c "ALTER USER sonar WITH ENCRYPTED PASSWORD 'admin123';"
   sudo -i -u postgres psql -c "CREATE DATABASE sonarqube OWNER sonar;"
   sudo -i -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE sonarqube to sonar;"
   systemctl restart  postgresql
   #systemctl status -l   postgresql
   netstat -tulpena | grep postgres
   sudo mkdir -p /sonarqube/
   cd /sonarqube/
   sudo curl -O https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-8.3.0.34182.zip
   sudo apt-get install zip -y
   sudo unzip -o sonarqube-8.3.0.34182.zip -d /opt/
   sudo mv /opt/sonarqube-8.3.0.34182/ /opt/sonarqube
   sudo groupadd sonar
   sudo useradd -c "SonarQube - User" -d /opt/sonarqube/ -g sonar sonar
   sudo chown sonar:sonar /opt/sonarqube/ -R
   cp /opt/sonarqube/conf/sonar.properties /root/sonar.properties_backup
   cat <<EOT> /opt/sonarqube/conf/sonar.properties
   sonar.jdbc.username=sonar
   sonar.jdbc.password=admin123
   sonar.jdbc.url=jdbc:postgresql://localhost/sonarqube
   sonar.web.host=0.0.0.0
   sonar.web.port=9000
   sonar.web.javaAdditionalOpts=-server
   sonar.search.javaOpts=-Xmx512m -Xms512m -XX:+HeapDumpOnOutOfMemoryError
   sonar.log.level=INFO
   sonar.path.logs=logs
   EOT
   
   cat <<EOT> /etc/systemd/system/sonarqube.service
   [Unit]
   Description=SonarQube service
   After=syslog.target network.target
   
   [Service]
   Type=forking
   
   ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
   ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop
   
   User=sonar
   Group=sonar
   Restart=always
   
   LimitNOFILE=65536
   LimitNPROC=4096
   
   
   [Install]
   WantedBy=multi-user.target
   EOT
   
   systemctl daemon-reload
   systemctl enable sonarqube.service
   #systemctl start sonarqube.service
   #systemctl status -l sonarqube.service
   apt-get install nginx -y
   rm -rf /etc/nginx/sites-enabled/default
   rm -rf /etc/nginx/sites-available/default
   cat <<EOT> /etc/nginx/sites-available/sonarqube
   server{
       listen      80;
       server_name sonarqube.groophy.in;
   
       access_log  /var/log/nginx/sonar.access.log;
       error_log   /var/log/nginx/sonar.error.log;
   
       proxy_buffers 16 64k;
       proxy_buffer_size 128k;
   
       location / {
           proxy_pass  http://127.0.0.1:9000;
           proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
           proxy_redirect off;
                 
           proxy_set_header    Host            \$host;
           proxy_set_header    X-Real-IP       \$remote_addr;
           proxy_set_header    X-Forwarded-For \$proxy_add_x_forwarded_for;
           proxy_set_header    X-Forwarded-Proto http;
       }
   }
   EOT
   ln -s /etc/nginx/sites-available/sonarqube /etc/nginx/sites-enabled/sonarqube
   systemctl enable nginx.service
   #systemctl restart nginx.service
   sudo ufw allow 80,9000,9001/tcp
   
   echo "System reboot in 30 sec"
   sleep 30
   reboot
```


## Install Jenkins

Use the following script in user-data
```
   #!/bin/bash
   sudo yum update –y
   sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
   sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
   #Removing old RPM Keys
   #sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
   #sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
   sudo yum upgrade -y
   sudo amazon-linux-extras install java-openjdk11 -y
   sudo yum install jenkins -y
   sudo systemctl enable jenkins
   sudo systemctl start jenkins
   
   # Installing Git
   sudo yum install git -y
   ###
   
   # Use The Amazon Linux 2 AMI When Launching The Jenkins VM/EC2 Instance
   # Instance Type: t2.medium or small minimum
   # Open Port (Security Group): 8080 
```
## Configure Nexus Repository 

Series of tutorial code snippets for use
#Maven publish tutorial steps
Publishing artifact to Nexus snapshot and release repo using maven.

1. Create a snapshot repo using nexus, or use default coming in out of the box. DEFAULT 
2. Create a release repo using nexus, or use default coming out of the box. DEFAULT
3. Create a group repo having both release, snapshot and other third party repos. or use default coming out of the box.
4. Download spring initializer project
5. Go settings.xml under <MAVEN_INSTALL_LOCATION>\apache-maven-3.6.0\conf or C:\Users\<USER_NAME>\.m2  or mkdir ~/.m2
6. Create/Move profiles named snapshot and release in settings.xml in `~/.m2` (can be done in pom.xml as well)
7. Add server user name and pwd in setting.xml (Encrypted recommended).
8. Edit pom.xml and add repository and snapshot repository in distribution management tag DEFAULT/DONE
9. Mark id should match in step 7 with server id of settings.xml, UPDATE NEXUS IP
10. Run the following `maven`/`mvn` commands to validate/package/deploy your app artifacts remotely
   - `mvn validate`   (validate the project is correct and all necessary information is available.)
   - `mvn compile`    (compile the source code of the project)
   - `mvn test`       (run tests using a suitable unit testing framework. These tests should not require the code be packaged or deployed.)
   - `mvn package`    (take the compiled code and package it in its distributable format, such as a WAR/JAR/EAR.)
   - `mvn verify`     (run any checks to verify the package is valid and meets quality criteria.)
   - `mvn install`    (install the package into the local repository, for use as a dependency in other projects locally.)
   - `mvn deploy`     (done in an integration or release environment, copies the final package to the remote/SNAPSHOT repository 
                      for sharing with other developers and projects.)

11. Change the version from 1.0-Snapshot to 1.0
12. Run `mvn deploy` to deploy to Snapshot Repo or `mvn clean deploy -P release`, to deploy it to Release Repo

## Maven Lifecycle Phases
- https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html#a-build-lifecycle-is-made-up-of-phases


# The Complete CI :

# Jenkins Maven Nexus Runbook

## Install M3 in Maven tools

Goto Manage Jenkins > Tools > Maven 

![](/Users/showmikbose/Library/Application%20Support/marktext/images/2023-07-23-18-37-21-image.png) 



## Create Jenkins Pipeline Project

Use following Script

    pipeline {
     agent any
    
      tools {
        // Install the Maven version configured as "M3" and add it to the path.
        maven "M3"
      }
     
      stages{
        stage("Checkout"){
            steps{
              git branch: 'maven-nexus', url: 'https://github.com/showmikb/eagles-batch-devops-projects.git'
            }
        }
      }
    
    }

### Run a Maven Build

```
pipeline {
 agent any

  tools {
    // Install the Maven version configured as "M3" and add it to the path.
    maven "M3"
  }
 
  stages{
    stage("Checkout"){
        steps{
          git branch: 'maven-nexus', url: 'https://github.com/showmikb/eagles-batch-devops-projects.git'
        }
    }


    stage('Build') {
        steps {
            dir('JavaWebApp') {
                sh 'mvn clean package'
            }
        }
    }

  }
}
```

### Now add a post block

```
pipeline {
    agent any

    tools {
        // Install the Maven version configured as "M3" and add it to the path.
        maven "M3"
    }

    stages {
        stage("Checkout") {
            steps {
                git branch: 'maven-nexus', url: 'https://github.com/showmikb/eagles-batch-devops-projects.git'
            }
        }

        stage('Build') {
            steps {
                dir('JavaWebApp') {
                    sh 'mvn clean package'
                }
            }
        }
    }

    post {
        always {
            // Archive the generated artifacts
            archiveArtifacts 'JavaWebApp/target/*.jar'
        }
    }
}

```

### Now add Test Stages

```
pipeline {
    agent any

    tools {
        // Install the Maven version configured as "M3" and add it to the path.
        maven "M3"
    }

    stages {
        stage("Checkout") {
            steps {
                git branch: 'maven-nexus', url: 'https://github.com/showmikb/eagles-batch-devops-projects.git'
            }
        }

        stage('Build') {
            steps {
                dir('JavaWebApp') {
                    sh 'mvn clean package'
                }
            }
            post {
                always {
                    // Archive the generated artifacts
                    archiveArtifacts 'JavaWebApp/target/*.jar'
                }
            }
        }
        stage('Unit Test'){
            steps {
                dir('JavaWebApp') {
                    sh 'mvn test'
                }
            }
        }
        
        stage('Integration Test'){
            steps {
                dir('JavaWebApp') {
                    sh 'mvn verify -DskipUnitTests'
                }
            }
        }    
        
    }
}

```

### Now add Checkstyle Scan

    pipeline {
        agent any
    
        tools {
            // Install the Maven version configured as "M3" and add it to the path.
            maven "M3"
        }
    
        stages {
            stage("Checkout") {
                steps {
                    git branch: 'maven-nexus', url: 'https://github.com/showmikb/eagles-batch-devops-projects.git'
                }
            }
    
            stage('Build') {
                steps {
                    dir('JavaWebApp') {
                        sh 'mvn clean package'
                    }
                }
                post {
                    always {
                        // Archive the generated artifacts
                        archiveArtifacts 'JavaWebApp/target/*.jar'
                    }
                }
            }
            stage('Unit Test'){
                steps {
                    dir('JavaWebApp') {
                        sh 'mvn test'
                    }
                }
            }
            
            stage('Integration Test'){
                steps {
                    dir('JavaWebApp') {
                        sh 'mvn verify -DskipUnitTests'
                    }
                }
            }    
            
            stage ('Checkstyle Code Analysis'){
                steps {
                    dir('JavaWebApp') {
                        sh 'mvn checkstyle:checkstyle'
                    }
                }
                post {
                    success {
                        echo 'Generated Analysis Result'
                    }
                }
            }
            
        }
    }
    

### Now do the Sonar Scan

```
pipeline {
    agent any

    tools {
        // Install the Maven version configured as "M3" and add it to the path.
        maven "M3"
    }

    stages {
        stage("Checkout") {
            steps {
                git branch: 'maven-nexus', url: 'https://github.com/showmikb/eagles-batch-devops-projects.git'
            }
        }

        stage('Build') {
            steps {
                dir('JavaWebApp') {
                    sh 'mvn clean package'
                }
            }
            post {
                always {
                    // Archive the generated artifacts
                    archiveArtifacts 'JavaWebApp/target/*.jar'
                }
            }
        }
        stage('Unit Test'){
            steps {
                dir('JavaWebApp') {
                    sh 'mvn test'
                }
            }
        }
        
        stage('Integration Test'){
            steps {
                dir('JavaWebApp') {
                    sh 'mvn verify -DskipUnitTests'
                }
            }
        }    
        
        stage ('Checkstyle Code Analysis'){
            steps {
                dir('JavaWebApp') {
                    sh 'mvn checkstyle:checkstyle'
                }
            }
            post {
                success {
                    echo 'Generated Analysis Result'
                }
            }
        }
        
        
        stage('SonarQube Scan') {
            steps {
                dir('JavaWebApp') {
                    sh """mvn sonar:sonar \
                          -Dsonar.projectKey=JavaWebApplication \
                          -Dsonar.host.url=http://3.110.171.226:9000 \
                          -Dsonar.login=df3865db3dc59a7788c09e04fa03090c1e0c9f4f"""
                }
            }
        }
    }
}

```



### Finally Deploy To Nexus

```
pipeline {
    agent any

    tools {
        // Install the Maven version configured as "M3" and add it to the path.
        maven "M3"
    }

    stages {
        stage("Checkout") {
            steps {
                git branch: 'maven-nexus', url: 'https://github.com/showmikb/eagles-batch-devops-projects.git'
            }
        }

        stage('Build') {
            steps {
                dir('JavaWebApp') {
                    sh 'mvn clean package'
                }
            }
            post {
                always {
                    // Archive the generated artifacts
                    archiveArtifacts 'JavaWebApp/target/*.jar'
                }
            }
        }
        stage('Unit Test'){
            steps {
                dir('JavaWebApp') {
                    sh 'mvn test'
                }
            }
        }
        
        stage('Integration Test'){
            steps {
                dir('JavaWebApp') {
                    sh 'mvn verify -DskipUnitTests'
                }
            }
        }    
        
        stage ('Checkstyle Code Analysis'){
            steps {
                dir('JavaWebApp') {
                    sh 'mvn checkstyle:checkstyle'
                }
            }
            post {
                success {
                    echo 'Generated Analysis Result'
                }
            }
        }
        
        
        stage('SonarQube Scan') {
            steps {
                dir('JavaWebApp') {
                    sh """mvn sonar:sonar \
                          -Dsonar.projectKey=JavaWebApplication \
                          -Dsonar.host.url=http://3.110.171.226:9000 \
                          -Dsonar.login=df3865db3dc59a7788c09e04fa03090c1e0c9f4f"""
                }
            }
        }
        
        stage('Nexus Deploy'){
            steps {
                dir('JavaWebApp') {
                        sh "mvn deploy -s ../settings.xml"
                }
            }
        }
    }
}

```



