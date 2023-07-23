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


