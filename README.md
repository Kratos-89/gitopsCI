# Continuous Integration with Jenkins

## Overview
This repository implements **Continuous Integration (CI)** using **Jenkins**, ensuring automated builds, testing, and artifact management for applications before deployment.

## Workflow Architecture
- **Jenkins Pipeline** – Automates code compilation, testing, and artifact generation.
- **Build and Test Steps** – Runs unit tests and performs static code analysis.
- **Docker Build & Push** – Containerizes the application and pushes images to a registry (Docker Hub, AWS ECR, etc.).
- **Artifact Management** – Stores and retrieves build artifacts for further deployment.
- **Trigger Mechanism** – Builds are triggered automatically on code commits or manually via Jenkins UI.

## Repository Structure
```
📂 jenkins/
 ├── 📄 Jenkinsfile  # Jenkins pipeline definition
📂 src/
 ├── 📄 application_code/  # Source code for the application
📂 tests/
 ├── 📄 unit_tests/  # Unit tests for the application
📄 Dockerfile  # Dockerfile for building the application container
📄 README.md  # Documentation
```

## Prerequisites
- Jenkins installed and configured
- Docker installed for local testing
- AWS, Docker Hub, or another container registry (if using Docker images)
- Required Jenkins plugins: **Pipeline, Docker Pipeline, Git Plugin**

## Jenkins CI Pipeline
### Step 1: Define the Jenkinsfile (`jenkins/Jenkinsfile`)
```groovy
pipeline{
  agent any

  parameters{
    string(name: 'DOCKER_TAG', defaultValue: 'latest', description: 'This is to keep track of the latest docker image tag.')
    //This can help us to pass the tag as a parameter during the build process.
  }

  environment{
    SCANNER_HOME = tool 'Sonar-Scanner' //Configure the sonarqube server into the system section of jenkins.
    //This actually points to the sonarqube server default shell path.
  }

  tools{
    maven 'maven3' 
    //maven is installed as tool and the local machine doesn't have maven, so we need configure the maven tool with the specific version and name maven3
  }
  stages{
    
    stage('Clean the workspace'){
      //This will clean the workspace to avoid any issues.
      steps{
        cleanWs()
      }
    }

    stage('Git Checkout'){
      //Use the pipeline syntax git, add the url, brach and the credentials to generate the below code.
      steps{
          git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/Kratos-89/gitopsCI.git'
      }
    }

    stage('Compile Java Code'){
      steps{
        //Assuming that the maven tool is configured.
        sh "mvn compile"
      }
    }

    stage('Test Compiled Code'){
      steps{
        sh "mvn test -DskipTests=true" 
        //Here the test cases are skipped because the java code has some issues.
      }
    }

    stage('File System Scan'){
      // Here the dependencies(pom.xml) scan is done to ensure that there is no dependency vulnerability or issues using trivy
      //Trivy is directly installed into the jenkins machine, because jenkins does not have a specific plugin for it yet.
      steps{
        sh "trivy fs --format table -o fs.html ."
        //This command scans the file system, generates the output in table format and stores it into fs.html file.
        //The '.' in the end mentions the path which needs to be scanned(Mention the path which contains the pom.xml file).
      }
    }

    stage('Sonar Analysis'){
      //Use the pipeline synatx -> withSonarQubeEnv
      steps{
        withSonarQubeEnv("sonarqube-server"){
        //specify the correct name of the sonarqube server added into jenkins.
        sh ''' $SCANNER_HOME/bin/sonar-scanner  -Dsonar.project=bankapp -Dsonar.projectKey=bank -Dsonar.java.binaries=target'''
        //This command gets into sonarQube server and executes the sonarqube-scanner executable.
        //  -Dsonar.project=bankapp \      # Project name in SonarQube UI  
        //  -Dsonar.projectKey=bank \      # Unique ID for the project  
        //  -Dsonar.java.binaries=target   # Path to compiled Java binaries (.class files) 
      }
      }
    }

    stage('Build & Publish'){
      //Before this step add the url of the release and snapshot nexus repositories into pom.xml file.
      //Make sure to change the deployment poilcy to "Allow redeploy" in nexus.
      //Configure the nexus server's credentials using config file global maven settings plugin
      steps{
        //Use pipeline synatx "withMaven:"
        withMaven(globalMavenSettingsConfig:'maven-settings-config',jdk:'', maven:'maven3',mavenSettingsConfig:'',traceability:true){
          sh 'mvn deploy -DskipTests=true'
        }
      }
    }

    stage('Docker Build'){
      steps{
        //The jenkins machine should have docker installed to execute the commands.
        script{
          //Use pipeline syntax "withDockerRegistry:"
          withDockerRegistry(credentialsId: 'doc-cred'){
            sh "docker build -t docravin/gitopsbankapp:${params.DOCKER_TAG} ."
            //Builds the docker image from the files in the '.'(current directory) and pushes to our docker hub. 
          }
        }
      }
    }

    stage('Docker Image Scan'){
      steps{
        //To scan the generated docker image.
        sh "trivy image --format=table -o dimage.html"  
      }
    }

    stage('Docker Push'){
      steps{
        script{
          withDockerRegistry(credentialsId: 'doc-cred'){
            sh "docker push docravin/gitopsbankapp:${params.DOCKER_TAG}"
          }
        }
      }
    }

    //Before this stage add the access token of gitopsCD repo.
    stage('Update the YAML files in the gitopsCD Repo'){
      //The purpose of this stage is to update the docker image's latest tag in the deployment file of the app. 
      steps{
        script{
          withCredentials([gitUsernamePassword(credentialsId: 'git-cred', gitToolName : 'Default')]){
            sh '''
            #Clone the repo
            git clone https://github.com/Kratos-89/gitops-CD.git
            cd gitops-CD

            #List files 
            ls -l bankapp

            #Get the absolute path
            repo_dir=$(pwd)

            #The below command searches the tag and updates it to the latest one.
            sed -i "s|image: docravin/gitopsbankapp:.*|image: docravin/gitopsbankapp:${DOCKER_TAG}|" ${repo_dir}/bankapp/bankapp.yaml
            '''
            //The tag changes are stored locally.
            
            //Confirm the change
            sh ''' 
            echo "Updated the deployment file"
            cat gitops-CD/bankapp/bankapp.yaml
            '''

            //Configure Git for committing changes and pushing.
            sh '''
            cd gitops-CD
            git config user.email "jenkins@cicd.com"
            git config user.name "jenkins" 
            '''

            // Commit and push the updated file.
            sh '''
              cd gitops-CD
              ls
              git add bankapp/bankapp.yaml
              git commit -m "Updated the image tag to ${Docker_TAG}" 
              git push origin main
            '''
          }
        }
      }
    }
    
  }
}
```

### Step 2: Configure Jenkins Pipeline Job
1. Open Jenkins dashboard and create a **New Item**.
2. Select **Pipeline** and provide a project name.
3. Under **Pipeline Definition**, select **Pipeline Script from SCM**.
4. Set **SCM** to Git and provide the repository URL.
5. Specify `jenkins/Jenkinsfile` as the script path.
6. Save and trigger the build.

### Step 3: Monitor Pipeline Execution
View the pipeline execution in Jenkins UI or check logs via:
```bash
jenkins-cli console -s http://your-jenkins-url
```

### Step 4: Download Artifacts (If Needed)
Artifacts from the CI run can be retrieved from Jenkins.

## Key Features
✔ **Automated Build and Test** – Ensures code quality and reliability.

✔ **Dockerized Workflows** – Supports containerized application builds.

✔ **Artifact Management** – Efficient storage and retrieval of compiled binaries.

✔ **Integration with CD Pipelines** – Can be extended to trigger ArgoCD deployments.


