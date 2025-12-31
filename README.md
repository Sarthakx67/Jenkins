# Jenkins Pipeline Complete Guide - RoboShop Project

## Table of Contents
1. [Overview](#overview)
2. [Jenkins Architecture](#jenkins-architecture)
3. [Core Concepts](#core-concepts)
4. [Project Structure](#project-structure)
5. [File-by-File Analysis](#file-by-file-analysis)
6. [Shared Library Deep Dive](#shared-library-deep-dive)
7. [Pipeline Flow Visualization](#pipeline-flow-visualization)
8. [Complete Examples](#complete-examples)
9. [Best Practices](#best-practices)
10. [Troubleshooting](#troubleshooting)

---

## Overview

This Jenkins setup implements a **CI/CD pipeline** for the RoboShop microservices project with the following capabilities:

- **Infrastructure as Code**: Automated VPC provisioning via Terraform
- **Multi-Application Support**: Node.js and Java applications (VM & EKS)
- **Shared Library Pattern**: Reusable pipeline code across multiple components
- **Artifact Management**: Integration with Nexus Repository
- **Environment Promotion**: Automated DEV deployment with manual approval for higher environments

**Key Components:**
- Jenkins master/agent architecture
- Terraform infrastructure automation
- Shared Jenkins libraries (custom functions)
- Multi-stage CI/CD pipelines
- Nexus artifact repository integration

---

## Jenkins Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Jenkins Master                        │
│                                                          │
│  ┌────────────────────────────────────────────────┐    │
│  │         Jenkins Shared Library                 │    │
│  │  (vars/ directory - reusable functions)        │    │
│  │  • pipelineDecision.groovy                     │    │
│  │  • nodeJSVMCI.groovy                           │    │
│  │  • javaVMCI.groovy                             │    │
│  └────────────────────────────────────────────────┘    │
│                                                          │
│  ┌────────────────────────────────────────────────┐    │
│  │         Pipeline Jobs                          │    │
│  │  • Infrastructure (Jenkinsfile-vpc01)          │    │
│  │  • Component CI jobs (per microservice)        │    │
│  │  • Component CD jobs (deployment)              │    │
│  └────────────────────────────────────────────────┘    │
└──────────────────┬───────────────────────────────────────┘
                   │
                   │ Job Distribution
                   │
      ┌────────────┴─────────────┐
      │                          │
┌─────▼──────┐          ┌────────▼──────┐
│  Agent-1   │          │   Agent-2     │
│            │          │               │
│  Builds &  │          │  Builds &     │
│  Tests     │          │  Tests        │
└────────────┘          └───────────────┘
      │                          │
      │                          │
      └──────────┬───────────────┘
                 │
                 │ Artifacts
                 │
        ┌────────▼─────────┐
        │  Nexus Repository│
        │                  │
        │  Stores built    │
        │  artifacts (.zip)│
        └──────────────────┘
                 │
                 │ Deployment
                 │
        ┌────────▼─────────┐
        │  Target Servers  │
        │  (DEV/PROD)      │
        └──────────────────┘
```

---

## Core Concepts

### 1. **Jenkins Pipeline**
Defines the entire CI/CD process as code (Jenkinsfile). Two types:
- **Declarative Pipeline**: Structured, easier syntax (used in this project)
- **Scripted Pipeline**: More flexible but complex

### 2. **Agent**
A machine/container where Jenkins executes pipeline stages.

```groovy
agent { label 'agent-1' }  // Runs on node with label 'agent-1'
agent { node { label 'AGENT-1' } }  // Alternative syntax
```

**Why use agents?**
- **Isolation**: Different environments for different jobs
- **Scalability**: Distribute workload across multiple machines
- **Resource Management**: Heavy builds don't impact Jenkins master

### 3. **Stages and Steps**
- **Stage**: Logical grouping of steps (Build, Test, Deploy)
- **Step**: Single task (sh command, script execution)

```groovy
stage('Build') {
    steps {
        sh 'npm install'
        sh 'npm run build'
    }
}
```

### 4. **Post Actions**
Code that runs after pipeline execution:
- `always`: Runs regardless of result
- `success`: Only on successful completion
- `failure`: Only on failure
- `unstable`: When tests fail but build succeeds

### 5. **Environment Variables**
Variables accessible throughout the pipeline:

```groovy
environment {
    USER = 'Sarthak'
    packageVersion = ''
}
```

**Accessing**: `${env.USER}` or `$USER`

### 6. **Parameters**
User inputs when manually triggering builds:
- `string`: Text input
- `boolean`: Checkbox
- `choice`: Dropdown menu
- `password`: Hidden text field

### 7. **Jenkins Shared Library**
Reusable code stored in Git repository, available to all pipelines.

**Structure:**
```
jenkins-shared-library/
├── vars/
│   ├── pipelineDecision.groovy
│   ├── nodeJSVMCI.groovy
│   └── javaVMCI.groovy
└── README.md
```

**Usage in Jenkinsfile:**
```groovy
@Library('shared-library-name') _
pipelineDecision(application: 'nodeJSVM', component: 'cart')
```

### 8. **Credentials Management**
Securely store sensitive data (passwords, SSH keys, API tokens).

```groovy
environment {
    AUTH = credentials('ssh-auth')  // Loads credential ID 'ssh-auth'
}
```

### 9. **Artifact Repository (Nexus)**
Central storage for build artifacts (JAR, ZIP, WAR files).

**Benefits:**
- Version control for artifacts
- Reusable across environments
- Audit trail of deployments

### 10. **Downstream Jobs**
Triggering other Jenkins jobs from within a pipeline.

```groovy
build job: "../${component}-deploy", parameters: [...]
```

---

## Project Structure

```
jenkins-project/
├── Jenkinsfile-vpc01          # Infrastructure provisioning pipeline
├── Jenkinsfile.bkp            # Comprehensive example (all features demo)
└── vars/                      # Shared library functions
    ├── pipelineDecision.groovy    # Router - decides which pipeline to use
    ├── nodeJSVMCI.groovy          # Node.js VM-based CI pipeline
    └── javaVMCI.groovy            # Java VM-based CI pipeline
```

---

## File-by-File Analysis

### **1. Jenkinsfile-vpc01 - Infrastructure Pipeline**

**Purpose:** Automates VPC infrastructure provisioning using Terraform.

```groovy
pipeline{
    agent { label 'agent-1' }

    stages{
        stage('downloading git repo...') {
            steps{
                sh '''
                ls -l
                cd /home/ec2-user
                git clone https://github.com/Sarthakx67/RoboShop-Infra-Standard.git
                '''
            }
        }
        stage('terraform init') {
            steps{
                sh '''
                    ls -l
                    cd /home/ec2-user/RoboShop-Infra-Standard/01-vpc
                    git pull
                    terraform init
                '''
            }
        }
        stage('terraform apply') {  # Note: This should be 'apply', not 'init' again
            steps{
                sh '''
                    cd /home/ec2-user/RoboShop-Infra-Standard/01-vpc
                    git pull
                    terraform apply -auto-approve
                '''
            }
        }
    }
    post{
        always {
            echo 'this block will always run whether job is success or not'
        }
        success {
            echo 'this block will run when job is success'
        }
        failure {
            echo 'this block will always run when job is unsuccessful'
        }
    }
}
```

#### **Stage-by-Stage Breakdown:**

**Stage 1: Downloading Git Repo**
```groovy
stage('downloading git repo...') {
    steps{
        sh '''
        ls -l
        cd /home/ec2-user
        git clone https://github.com/Sarthakx67/RoboShop-Infra-Standard.git
        '''
    }
}
```
- **Purpose**: Clone Terraform infrastructure code
- **sh ''': Multi-line shell script execution
- **ls -l**: Lists directory contents (debugging)
- **Issue**: Cloning every time is inefficient (should check if exists first)

**Stage 2: Terraform Init**
```groovy
stage('terraform init') {
    steps{
        sh '''
            ls -l
            cd /home/ec2-user/RoboShop-Infra-Standard/01-vpc
            git pull
            terraform init
        '''
    }
}
```
- **git pull**: Gets latest changes (good if repo exists)
- **terraform init**: Initializes Terraform (downloads providers, modules)
- **Downloads**: AWS provider, state backend configuration

**Stage 3: Terraform Apply**
```groovy
stage('terraform apply') {
    steps{
        sh '''
            cd /home/ec2-user/RoboShop-Infra-Standard/01-vpc
            git pull
            terraform apply -auto-approve
        '''
    }
}
```
- **terraform apply**: Creates/updates infrastructure
- **-auto-approve**: Skips manual confirmation (use carefully!)
- **Missing**: `terraform plan` stage for validation

#### **Post Actions:**
```groovy
post{
    always {
        echo 'this block will always run whether job is success or not'
    }
    success {
        echo 'this block will run when job is success'
    }
    failure {
        echo 'this block will always run when job is unsuccessful'
    }
}
```

**Use Cases:**
- `always`: Cleanup, notifications
- `success`: Send success email, trigger deployment
- `failure`: Send alert, rollback changes

#### **Issues in Current Pipeline:**

1. **Duplicate Stage Name**: Two stages named 'terraform init'
2. **No Error Handling**: If git clone fails (repo exists), pipeline breaks
3. **No Validation**: Missing `terraform plan` before apply
4. **Hardcoded Paths**: `/home/ec2-user` not portable
5. **No State Management**: No workspace cleanup

#### **Improved Version:**

```groovy
pipeline{
    agent { label 'agent-1' }
    
    environment {
        WORKSPACE_DIR = '/home/ec2-user/RoboShop-Infra-Standard'
    }

    stages{
        stage('Checkout') {
            steps{
                script {
                    sh '''
                        if [ -d "${WORKSPACE_DIR}" ]; then
                            cd ${WORKSPACE_DIR}
                            git pull
                        else
                            cd /home/ec2-user
                            git clone https://github.com/Sarthakx67/RoboShop-Infra-Standard.git
                        fi
                    '''
                }
            }
        }
        stage('Terraform Init') {
            steps{
                sh '''
                    cd ${WORKSPACE_DIR}/01-vpc
                    terraform init
                '''
            }
        }
        stage('Terraform Plan') {
            steps{
                sh '''
                    cd ${WORKSPACE_DIR}/01-vpc
                    terraform plan -out=tfplan
                '''
            }
        }
        stage('Approve') {
            input {
                message "Review plan and approve?"
                ok "Yes, apply changes"
            }
            steps {
                echo "Approved by ${env.BUILD_USER}"
            }
        }
        stage('Terraform Apply') {
            steps{
                sh '''
                    cd ${WORKSPACE_DIR}/01-vpc
                    terraform apply tfplan
                '''
            }
        }
    }
    
    post{
        always {
            echo 'Pipeline completed'
            cleanWs()  // Clean workspace
        }
        success {
            echo 'Infrastructure provisioned successfully'
            // Send success notification
        }
        failure {
            echo 'Pipeline failed - check logs'
            // Send alert to team
        }
    }
}
```

---

### **2. Jenkinsfile.bkp - Comprehensive Example**

**Purpose:** Demonstrates ALL Jenkins features - a learning reference.

#### **Key Sections:**

**A. Agent Configuration**
```groovy
agent { node { label 'AGENT-1' } }
```

**B. Options**
```groovy
options {
    timeout(time: 1, unit: 'HOURS')  // Pipeline timeout
}
```
- Prevents pipelines from running indefinitely
- `unit` options: SECONDS, MINUTES, HOURS, DAYS

**C. Triggers (Commented)**
```groovy
// triggers {
//     cron('* * * * *')  // Every minute
// }
```

**Cron Syntax:** `minute hour day month weekday`
- `0 0 * * *`: Daily at midnight
- `0 9 * * 1-5`: Weekdays at 9 AM
- `*/15 * * * *`: Every 15 minutes

**D. Environment Variables**
```groovy
environment { 
    USER = 'Sarthak'
}
```
- **Global**: Available in all stages
- Access via `${env.USER}` or `$USER`

**E. Parameters**
```groovy
parameters {
    string(name: 'PERSON', defaultValue: 'Mr Jenkins', description: 'Who should I say hello to?')
    text(name: 'BIOGRAPHY', defaultValue: '', description: 'Enter some information')
    booleanParam(name: 'TOGGLE', defaultValue: true, description: 'Toggle this value')
    choice(name: 'CHOICE', choices: ['One', 'Two', 'Three'], description: 'Pick something')
    password(name: 'PASSWORD', defaultValue: 'SECRET', description: 'Enter a password')
}
```

**Usage in Stages:**
```groovy
echo "Hello ${params.PERSON}"
echo "Choice: ${params.CHOICE}"
```

**F. Basic Stages**
```groovy
stage('Build') {
    steps {
        echo 'Building..'
        sh '''
            ls -ltr
            pwd
            echo "Hello from GitHub Push webhook event"
            printenv  // Print all environment variables
        '''
    }
}
```

**G. Credentials Usage**
```groovy
stage('Example') {
    environment { 
        AUTH = credentials('ssh-auth')  // Loads credential from Jenkins
    }
    steps {
        sh 'printenv'  // AUTH_USR and AUTH_PSW will be available
    }
}
```

**How credentials work:**
- Stored securely in Jenkins
- Referenced by ID ('ssh-auth')
- Exposed as environment variables
- `AUTH_USR`: Username
- `AUTH_PSW`: Password

**H. Input Stage (Manual Approval)**
```groovy
stage('Input') {
    input {
        message "Should we continue?"
        ok "Yes, we should."
        submitter "alice,bob"  // Only these users can approve
        parameters {
            string(name: 'PERSON', defaultValue: 'Mr Jenkins', description: 'Who should I say hello to?')
        }
    }
    steps {
        echo "Hello, ${PERSON}, nice to meet you."
    }
}
```

**Use Cases:**
- Production deployments
- Sensitive operations
- Cost-expensive actions

**I. Conditional Execution**
```groovy
stage('PROD Deploy'){
    when {
        environment name: 'USER', value: 'Sarthak'
    }
    steps{
        echo "deploying to PROD"
    }
}
```

**Other `when` conditions:**
```groovy
when { branch 'master' }  // Only on master branch
when { environment name: 'ENV', value: 'prod' }
when { expression { return params.DEPLOY == true } }
```

**J. Parallel Execution**
```groovy
stage('Parallel Stage') {
    parallel {
        stage('Branch A') {
            steps {
                echo "On Branch A"
                sh 'sleep 10'
            }
        }
        stage('Branch B') {
            steps {
                echo "On Branch B"
                sh 'sleep 10'
            }
        }
        stage('Branch C') {
            stages {
                stage('Nested 1') {
                    steps {
                        echo "In stage Nested 1 within Branch C"
                        sh 'sleep 10'
                    }
                }
                stage('Nested 2') {
                    steps {
                        echo "In stage Nested 2 within Branch C"
                        sh 'sleep 10'
                    }
                }
            }
        }
    }
}
```

**Execution Pattern:**
```
Time: 0s ────────────► 10s
      │
      ├─ Branch A ─────────► (parallel)
      ├─ Branch B ─────────► (parallel)
      └─ Branch C
         ├─ Nested 1 ──────► (sequential)
         └─ Nested 2 ──────────────────► (sequential)
```

**Use Cases:**
- Running tests across different browsers
- Building multiple artifacts simultaneously
- Deploying to multiple regions

---

### **3. vars/pipelineDecision.groovy - Pipeline Router**

**Purpose:** Central decision point that routes to appropriate pipeline based on application type.

```groovy
#!groovy

def decidePipeline(Map configMap){
    application = configMap.get("application")
    
    switch(application) {
        case 'nodeJSVM':
            echo "application is Node JS and VM based"
            nodeJSVMCI(configMap)
            break
        case 'nodeJSEKS':
            echo "application is Node JS and EKS based"
            nodeJSEKS(configMap)
            break
        case 'JavaVM':
            echo "application is Java and VM based"
            javaVMCI(configMap)
            break
        case 'javaEKS':
            echo "application is Java and EKS based"
            javaEKS(configMap)
            break
        default:
            error "Unrecognised application"
            break
    }
}
```

#### **Breakdown:**

**Function Signature:**
```groovy
def decidePipeline(Map configMap)
```
- `def`: Groovy function definition
- `Map configMap`: Accepts a map (dictionary) as parameter

**Map Operations:**
```groovy
application = configMap.get("application")
```
- Retrieves value for key "application"
- Example: `configMap = [application: 'nodeJSVM', component: 'cart']`
- Returns: `'nodeJSVM'`

**Switch Statement:**
```groovy
switch(application) {
    case 'nodeJSVM':
        nodeJSVMCI(configMap)  // Calls another shared library function
        break
    default:
        error "Unrecognised application"  // Fails pipeline
}
```

**How It's Called (in microservice Jenkinsfile):**
```groovy
@Library('roboshop-shared-library') _

def configMap = [
    application: 'nodeJSVM',
    component: 'cart'
]

decidePipeline(configMap)
```

**Design Pattern:** Strategy Pattern
- Single entry point
- Routes to specialized pipelines
- Easy to add new application types
- Reduces code duplication

---

### **4. vars/nodeJSVMCI.groovy - Node.js CI Pipeline**

**Purpose:** Complete CI pipeline for Node.js applications on VMs.

```groovy
def call(Map configMap){
    def component = configMap.get("component")
    echo "component is : $component"
    
    pipeline {
        agent { node { label 'AGENT-1' } }
        
        environment{
            packageVersion = ''
        }
        
        stages {
            stage('Get version'){
                steps{
                    script{
                        def packageJson = readJSON(file: 'package.json')
                        packageVersion = packageJson.version
                        echo "version: ${packageVersion}"
                    }
                }
            }
            stage('Install dependencies') {
                steps {
                    sh 'npm install'
                }
            }
            stage('Unit test') {
                steps {
                    echo "unit testing is done here"
                }
            }
            stage('Sonar Scan') {
                steps {
                    echo "Sonar scan done"
                }
            }
            stage('Build') {
                steps {
                    sh 'ls -ltr'
                    sh "zip -r ${component}.zip ./* --exclude=.git --exclude=.zip"
                }
            }
            stage('SAST') {
                steps {
                    echo "SAST Done"
                    echo "package version: $packageVersion"
                }
            }
            stage('Publish Artifact') {
                steps {
                    nexusArtifactUploader(
                        nexusVersion: 'nexus3',
                        protocol: 'http',
                        nexusUrl: '172.31.86.20:8081/',
                        groupId: 'com.roboshop',
                        version: "$packageVersion",
                        repository: "${component}",
                        credentialsId: 'nexus-auth',
                        artifacts: [
                            [artifactId: "${component}",
                            classifier: '',
                            file: "${component}.zip",
                            type: 'zip']
                        ]
                    )
                }
            }
            stage('Deploy') {
                steps {
                    script{
                        echo "Deployment"
                        def params = [
                            string(name: 'version', value: "$packageVersion"),
                            string(name: 'environment', value: "dev")
                        ]
                        build job: "../${component}-deploy", wait: true, parameters: params
                    }
                }
            }
        }

        post{
            always{
                echo 'cleaning up workspace'
                //deleteDir()
            }
        }
    }
}
```

#### **Stage-by-Stage Analysis:**

**Stage 1: Get Version**
```groovy
stage('Get version'){
    steps{
        script{
            def packageJson = readJSON(file: 'package.json')
            packageVersion = packageJson.version
            echo "version: ${packageVersion}"
        }
    }
}
```

**Purpose:** Extract version from package.json for artifact versioning

**readJSON()**: Jenkins plugin function
- Parses JSON files
- Returns map/object
- Requires "Pipeline Utility Steps" plugin

**Example package.json:**
```json
{
  "name": "cart",
  "version": "1.0.5",
  "description": "Shopping cart microservice"
}
```

**Result:** `packageVersion = "1.0.5"`

**Stage 2: Install Dependencies**
```groovy
stage('Install dependencies') {
    steps {
        sh 'npm install'
    }
}
```
- Downloads Node.js packages from package.json
- Creates node_modules/ directory
- Required before testing/building

**Stage 3: Unit Test**
```groovy
stage('Unit test') {
    steps {
        echo "unit testing is done here"
    }
}
```

**Real Implementation Would Be:**
```groovy
sh 'npm test'
junit 'test-results/*.xml'  // Publish test results
```

**Stage 4: Sonar Scan**
```groovy
stage('Sonar Scan') {
    steps {
        echo "Sonar scan done"
    }
}
```

**Purpose:** Code quality analysis

**Real Implementation:**
```groovy
withSonarQubeEnv('SonarQube') {
    sh 'sonar-scanner'
}
```

**Checks:**
- Code smells
- Security vulnerabilities
- Code coverage
- Duplication

**Stage 5: Build**
```groovy
stage('Build') {
    steps {
        sh 'ls -ltr'
        sh "zip -r ${component}.zip ./* --exclude=.git --exclude=.zip"
    }
}
```

**Purpose:** Create deployable artifact

**zip command breakdown:**
- `zip -r`: Recursive (include subdirectories)
- `${component}.zip`: Output file (e.g., cart.zip)
- `./*`: All files in current directory
- `--exclude=.git`: Skip .git directory
- `--exclude=.zip`: Skip existing zip files

**Stage 6: SAST (Static Application Security Testing)**
```groovy
stage('SAST') {
    steps {
        echo "SAST Done"
        echo "package version: $packageVersion"
    }
}
```

**Tools:** Checkmarx, Fortify, Veracode

**Real Implementation:**
```groovy
sh 'snyk test'  // Snyk for dependency scanning
```

**Stage 7: Publish Artifact**
```groovy
stage('Publish Artifact') {
    steps {
        nexusArtifactUploader(
            nexusVersion: 'nexus3',
            protocol: 'http',
            nexusUrl: '172.31.86.20:8081/',
            groupId: 'com.roboshop',
            version: "$packageVersion",
            repository: "${component}",
            credentialsId: 'nexus-auth',
            artifacts: [
                [artifactId: "${component}",
                classifier: '',
                file: "${component}.zip",
                type: 'zip']
            ]
        )
    }
}
```

**Parameters Explained:**
- `nexusVersion`: Nexus version (nexus2/nexus3)
- `nexusUrl`: Nexus server address
- `groupId`: Maven-style group (com.roboshop)
- `version`: Artifact version (from package.json)
- `repository`: Nexus repository name (matches component)
- `credentialsId`: Jenkins credential ID for Nexus auth
- `artifactId`: Artifact identifier
- `file`: Path to artifact file
- `type`: Artifact type (zip, jar, war)

**Nexus Structure:**
```
Nexus Repository
└── cart/
    ├── 1.0.5/
    │   └── cart.zip
    ├── 1.0.4/
    │   └── cart.zip
    └── 1.0.3/
        └── cart.zip
```

**Benefits:**
- Version history
- Rollback capability
- Centralized storage
- Audit trail

**Stage 8: Deploy**
```groovy
stage('Deploy') {
    steps {
        script{
            echo "Deployment"
            def params = [
                string(name: 'version', value: "$packageVersion"),
                string(name: 'environment', value: "dev")
            ]
            build job: "../${component}-deploy", wait: true, parameters: params
        }
    }
}
```

**Purpose:** Trigger deployment job

**build job parameters:**
- `job`: Path to deployment job (e.g., "../cart-deploy")
- `wait`: Wait for job completion (true/false)
- `parameters`: Pass values to downstream job

**Deployment Job Structure:**
```
Jenkins/
├── CI/
│   └── cart-ci  (This pipeline)
└── CD/
    └── cart-deploy  (Triggered job)
```

**Downstream job receives:**
- `version`: "1.0.5"
- `environment`: "dev"

**Deployment job will:**
1. Download artifact from Nexus (version 1.0.5)
2. Deploy to DEV environment servers
3. Run smoke tests
4. Report status

---

### **5. vars/javaVMCI.groovy - Java CI Pipeline**

```groovy
def call(Map configMap){
    echo "This will run if application is Java and VM"
}
```

**Current State:** Placeholder (not implemented)

**Full Implementation Would Include:**
```groovy
def call(Map configMap){
    def component = configMap.get("component")
    
    pipeline {
        agent { node { label 'AGENT-1' } }
        
        stages {
            stage('Maven Build') {
                steps {
                    sh 'mvn clean package'
                }
            }
            stage('Unit Tests') {
                steps {
                    sh 'mvn test'
                    junit '**/target/surefire-reports/*.xml'
                }
            }
            stage('SonarQube') {
                steps {
                    withSonarQubeEnv('SonarQube') {
                        sh 'mvn sonar:sonar'
                    }
                }
            }
            stage('Publish to Nexus') {
                steps {
                    sh 'mvn deploy'
                }
            }
            stage('Deploy') {
                steps {
                    script {
                        build job: "../${component}-deploy"
                    }
                }
            }
        }
    }
}
```

---

## Shared Library Deep Dive

### **What is a Jenkins Shared Library?**

A Git repository containing reusable pipeline code that can be imported into any Jenkins pipeline.

### **Directory Structure:**
```
jenkins-shared-library/
├── vars/                  # Global variables (pipeline functions)
│   ├── pipelineDecision.groovy
│   ├── nodeJSVMCI.groovy
│   ├── nodeJSEKS.groovy
│   ├── javaVMCI.groovy
│   └── javaEKS.groovy
├── src/                   # Groovy classes (optional)
│   └── org/
│       └── roboshop/
│           └── Utils.groovy
└── resources/             # Non-Groovy files (optional)
    └── scripts/
        └── deploy.sh
```

### **How to Configure:**

**1. Add Library in Jenkins:**
```
Manage Jenkins → Configure System → Global Pipeline Libraries

Name: roboshop-shared-library
Default version: main
Retrieval method: Modern SCM
  └── Git
      └── Project Repository: https://github.com/user/jenkins-shared-library.git
```

**2. Use in Jenkinsfile:**
```groovy
@Library('roboshop-shared-library') _

def configMap = [
    application: 'nodeJSVM',
    component: 'cart'
]

decidePipeline(configMap)
```

**3. What Happens:**
```
Pipeline runs
    ↓
@Library loads shared library from Git
    ↓
decidePipeline() function is available
    ↓
Calls nodeJSVMCI(configMap)
    ↓
Full pipeline executes
```

### **Benefits:**

1. **DRY Principle**: Don't Repeat Yourself
2. **Centralized Updates**: Change once, apply everywhere
3. **Consistency**: Same process for all components
4. **Version Control**: Track changes to pipeline code
5. **Testing**: Test pipeline changes before production

### **Real-World Example:**

**Without Shared Library:**
```
cart/Jenkinsfile          (200 lines)
user/Jenkinsfile          (200 lines, mostly duplicate)
catalogue/Jenkinsfile     (200 lines, mostly duplicate)
shipping/Jenkinsfile      (200 lines, mostly duplicate)
payment/Jenkinsfile       (200 lines, mostly duplicate)
```

**With Shared Library:**
```
shared-library/vars/nodeJSVMCI.groovy  (200 lines)

cart/Jenkinsfile          (5 lines)
user/Jenkinsfile          (5 lines)
catalogue/Jenkinsfile     (5 lines)
shipping/Jenkinsfile      (5 lines)
payment/Jenkinsfile       (5 lines)
```

**Each microservice Jenkinsfile:**
```groovy
@Library('roboshop-shared-library') _
decidePipeline(application: 'nodeJSVM', component: 'cart')
```

---

## Pipeline Flow Visualization

### **Complete CI/CD Flow**

```
┌─────────────────────────────────────────────────────────────┐
│                   Developer Workflow                         │
└─────────────────────────────────────────────────────────────┘
                            │
                            │ git push
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                   GitHub Repository                          │
│                   (Microservice Code)                        │
└─────────────────────────────────────────────────────────────┘
                            │
                            │ Webhook trigger
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                    Jenkins Master                            │
│                                                              │
│  1. Loads Shared Library from Git                           │
│  2. Calls decidePipeline()                                  │
│  3. Routes to nodeJSVMCI()                                  │
└─────────────────────────────────────────────────────────────┘
                            │
                            │ Assigns to Agent
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                    Jenkins Agent-1                           │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Stage 1: Get Version                                │  │
│  │  • Read package.json                                 │  │
│  │  • Extract version: 1.0.5                            │  │
│  └──────────────────────────────────────────────────────┘  │
│                            ↓                                 │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Stage 2: Install Dependencies                       │  │
│  │  • npm install                                       │  │
│  │  • Downloads node_modules                            │  │
│  └──────────────────────────────────────────────────────┘  │
│                            ↓                                 │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Stage 3: Unit Tests                                 │  │
│  │  • npm test                                          │  │
│  │  • Generate test reports                             │  │
│  └──────────────────────────────────────────────────────┘  │
│                            ↓                                 │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Stage 4: Code Quality (SonarQube)                   │  │
│  │  • Static code analysis                              │  │
│  │  • Security vulnerability scan                        │  │
│  └──────────────────────────────────────────────────────┘  │
│                            ↓                                 │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Stage 5: Build Artifact                             │  │
│  │  • zip -r cart.zip ./*                               │  │
│  │  • Artifact: cart-1.0.5.zip                          │  │
│  └──────────────────────────────────────────────────────┘  │
│                            ↓                                 │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Stage 6: SAST Scan                                  │  │
│  │  • Security scanning of code                         │  │
│  │  • Dependency vulnerability check                     │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                            │
                            │ Upload artifact
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                   Nexus Repository                           │
│                                                              │
│  Repository: cart/                                          │
│  Version: 1.0.5                                             │
│  Artifact: cart.zip                                         │
│  Size: 25 MB                                                │
│  Uploaded: 2025-12-31 10:30:45                              │
└─────────────────────────────────────────────────────────────┘
                            │
                            │ Trigger deployment
                            ↓
┌─────────────────────────────────────────────────────────────┐
│              Downstream Deployment Job                       │
│              (cart-deploy)                                   │
│                                                              │
│  Parameters:                                                │
│  • version: 1.0.5                                           │
│  • environment: dev                                         │
│                                                              │
│  Steps:                                                     │
│  1. Download cart-1.0.5.zip from Nexus                      │
│  2. Stop existing service                                   │
│  3. Deploy new version                                      │
│  4. Start service                                           │
│  5. Health check                                            │
└─────────────────────────────────────────────────────────────┘
                            │
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                  DEV Environment                             │
│                                                              │
│  cart-service:1.0.5 [RUNNING]                               │
└─────────────────────────────────────────────────────────────┘
```

### **Infrastructure Pipeline Flow**

```
┌─────────────────────────────────────────────────────────────┐
│         Manual Trigger / Scheduled Trigger                   │
└─────────────────────────────────────────────────────────────┘
                            │
                            ↓
┌─────────────────────────────────────────────────────────────┐
│               Jenkins Master                                 │
│               Jenkinsfile-vpc01                              │
└─────────────────────────────────────────────────────────────┘
                            │
                            │ Assigns to Agent
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                   Jenkins Agent-1                            │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Stage 1: Checkout Code                              │  │
│  │  • Clone/Pull Terraform repo                         │  │
│  │  • Location: /home/ec2-user/RoboShop-Infra-Standard │  │
│  └──────────────────────────────────────────────────────┘  │
│                            ↓                                 │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Stage 2: Terraform Init                             │  │
│  │  • terraform init                                    │  │
│  │  • Download AWS provider                             │  │
│  │  • Initialize backend                                │  │
│  └──────────────────────────────────────────────────────┘  │
│                            ↓                                 │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Stage 3: Terraform Plan (Recommended)               │  │
│  │  • terraform plan                                    │  │
│  │  • Preview changes                                   │  │
│  └──────────────────────────────────────────────────────┘  │
│                            ↓                                 │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Stage 4: Terraform Apply                            │  │
│  │  • terraform apply -auto-approve                     │  │
│  │  • Create VPC, subnets, route tables                 │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                            │
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                      AWS Account                             │
│                                                              │
│  Resources Created:                                         │
│  • VPC: roboshop-dev                                        │
│  • Public Subnets: 2                                        │
│  • Private Subnets: 2                                       │
│  • Database Subnets: 2                                      │
│  • Internet Gateway: 1                                      │
│  • Route Tables: 3                                          │
│  • NAT Gateway: 0 (Testing mode)                            │
└─────────────────────────────────────────────────────────────┘
```

---

## Complete Examples

### **Example 1: Cart Microservice CI/CD**

**cart/Jenkinsfile:**
```groovy
@Library('roboshop-shared-library') _

def configMap = [
    application: 'nodeJSVM',
    component: 'cart'
]

decidePipeline(configMap)
```

**Execution:**
1. Developer pushes code to cart repository
2. GitHub webhook triggers Jenkins job
3. Jenkins loads shared library
4. Calls `decidePipeline(configMap)`
5. Routes to `nodeJSVMCI(configMap)`
6. Full CI pipeline executes on Agent-1
7. Artifact (cart-1.0.5.zip) uploaded to Nexus
8. Deployment job triggered for DEV environment

**Timeline:**
```
00:00 - Triggered by webhook
00:05 - Checkout code
00:10 - Get version (1.0.5)
00:15 - Install dependencies (npm install)
02:30 - Unit tests (npm test)
03:00 - SonarQube scan
04:30 - Build artifact (zip)
05:00 - SAST scan
05:30 - Publish to Nexus
06:00 - Trigger deployment
06:30 - Deployment complete ✓
```

### **Example 2: User Microservice with Different Version**

**user/Jenkinsfile:**
```groovy
@Library('roboshop-shared-library') _

def configMap = [
    application: 'nodeJSVM',
    component: 'user'
]

decidePipeline(configMap)
```

**Same pipeline, different component:**
- Artifact: user-2.1.3.zip (different version)
- Nexus repository: user/
- Deployment job: user-deploy

### **Example 3: Java-based Payment Service**

**payment/Jenkinsfile:**
```groovy
@Library('roboshop-shared-library') _

def configMap = [
    application: 'JavaVM',
    component: 'payment'
]

decidePipeline(configMap)
```

**Routes to:** `javaVMCI(configMap)`
- Maven build instead of npm
- JAR artifact instead of ZIP
- Different test framework

### **Example 4: Multi-Environment Deployment**

**Deployment Pipeline (cart-deploy/Jenkinsfile):**
```groovy
pipeline {
    agent { label 'AGENT-1' }
    
    parameters {
        string(name: 'version', description: 'Artifact version to deploy')
        choice(name: 'environment', choices: ['dev', 'qa', 'prod'], description: 'Target environment')
    }
    
    stages {
        stage('Download Artifact') {
            steps {
                script {
                    sh """
                        curl -u admin:password \
                        -o cart.zip \
                        http://nexus:8081/repository/cart/${params.version}/cart.zip
                    """
                }
            }
        }
        
        stage('Deploy to DEV') {
            when {
                expression { params.environment == 'dev' }
            }
            steps {
                sh 'ansible-playbook -i inventory/dev deploy.yml'
            }
        }
        
        stage('Deploy to QA') {
            when {
                expression { params.environment == 'qa' }
            }
            steps {
                input message: 'Approve QA deployment?', submitter: 'qa-team'
                sh 'ansible-playbook -i inventory/qa deploy.yml'
            }
        }
        
        stage('Deploy to PROD') {
            when {
                expression { params.environment == 'prod' }
            }
            steps {
                input message: 'Approve PROD deployment?', submitter: 'devops-lead'
                sh 'ansible-playbook -i inventory/prod deploy.yml'
            }
        }
        
        stage('Smoke Test') {
            steps {
                sh '''
                    sleep 10
                    curl -f http://cart-service:8080/health || exit 1
                '''
            }
        }
    }
    
    post {
        success {
            echo "Deployed cart version ${params.version} to ${params.environment}"
        }
        failure {
            echo "Deployment failed - rolling back"
            sh 'ansible-playbook -i inventory/${params.environment} rollback.yml'
        }
    }
}
```

---

## Best Practices

### **1. Pipeline Design**

✅ **DO:**
- Use declarative pipelines (easier to read)
- Keep pipelines DRY with shared libraries
- Use meaningful stage names
- Add proper error handling
- Implement proper logging

❌ **DON'T:**
- Hardcode credentials
- Use overly complex pipelines
- Skip testing stages
- Ignore failed tests

### **2. Security**

✅ **DO:**
```groovy
environment {
    CREDENTIALS = credentials('my-secret-id')
}
```

❌ **DON'T:**
```groovy
environment {
    PASSWORD = 'hardcoded-password'  // NEVER DO THIS
}
```

### **3. Error Handling**

✅ **DO:**
```groovy
try {
    sh 'risky-command'
} catch (Exception e) {
    echo "Error: ${e.message}"
    currentBuild.result = 'FAILURE'
}
```

### **4. Artifact Management**

✅ **DO:**
- Use semantic versioning (1.0.5)
- Store artifacts in Nexus/Artifactory
- Keep artifact history
- Tag Docker images with version

❌ **DON'T:**
- Use 'latest' tag in production
- Store artifacts on Jenkins server
- Skip versioning

### **5. Shared Library Structure**

✅ **DO:**
```
vars/
├── pipelineDecision.groovy    (Router)
├── nodeJSVMCI.groovy          (Specific implementation)
└── commonFunctions.groovy     (Utilities)
```

❌ **DON'T:**
```
vars/
└── everything.groovy  (1000 lines of mixed code)
```

### **6. Testing**

```groovy
stage('Tests') {
    parallel {
        stage('Unit Tests') {
            steps {
                sh 'npm test'
                junit 'test-results/*.xml'
            }
        }
        stage('Integration Tests') {
            steps {
                sh 'npm run integration-test'
            }
        }
        stage('Security Scan') {
            steps {
                sh 'snyk test'
            }
        }
    }
}
```

### **7. Notifications**

```groovy
post {
    success {
        slackSend(
            color: 'good',
            message: "Build ${env.BUILD_NUMBER} succeeded"
        )
    }
    failure {
        slackSend(
            color: 'danger',
            message: "Build ${env.BUILD_NUMBER} failed"
        )
        emailext(
            to: 'devops-team@company.com',
            subject: "Pipeline Failed: ${env.JOB_NAME}",
            body: "Check console output at ${env.BUILD_URL}"
        )
    }
}
```

### **8. Performance Optimization**

```groovy
options {
    buildDiscarder(logRotator(numToKeepStr: '10'))  // Keep only 10 builds
    disableConcurrentBuilds()  // Prevent parallel runs
    timeout(time: 1, unit: 'HOURS')  // Timeout after 1 hour
}
```

### **9. Git Best Practices**

```groovy
stage('Checkout') {
    steps {
        checkout([
            $class: 'GitSCM',
            branches: [[name: '*/main']],
            userRemoteConfigs: [[
                url: 'https://github.com/user/repo.git',
                credentialsId: 'github-credentials'
            ]]
        ])
    }
}
```

### **10. Documentation**

```groovy
// Pipeline: cart-ci
// Purpose: Build and test cart microservice
// Owner: DevOps Team
// Last Modified: 2025-12-31

pipeline {
    // Clear, documented stages
}
```

---

## Troubleshooting

### **Common Issues and Solutions**

#### **Issue 1: "No such file or directory"**
```
Error: cd /home/ec2-user/RoboShop-Infra-Standard/01-vpc: No such file or directory
```

**Solution:**
```groovy
sh '''
    if [ ! -d "/home/ec2-user/RoboShop-Infra-Standard" ]; then
        git clone https://github.com/Sarthakx67/RoboShop-Infra-Standard.git
    fi
    cd /home/ec2-user/RoboShop-Infra-Standard/01-vpc
    git pull
'''
```

#### **Issue 2: "Terraform not found"**
```
Error: terraform: command not found
```

**Solution:** Install Terraform on agent
```bash
sudo yum install -y terraform
```

Or use Docker agent with Terraform pre-installed:
```groovy
agent {
    docker {
        image 'hashicorp/terraform:latest'
        label 'docker-agent'
    }
}
```

#### **Issue 3: "Permission denied (publickey)"**
```
Error: Permission denied (publickey)
```

**Solution:** Add SSH credentials
```groovy
stages {
    stage('Checkout') {
        steps {
            git credentialsId: 'github-ssh-key', url: 'git@github.com:user/repo.git'
        }
    }
}
```

#### **Issue 4: "Nexus upload failed"**
```
Error: 401 Unauthorized
```

**Solution:** Check Nexus credentials
```
Manage Jenkins → Credentials → Add Credentials
Kind: Username with password
ID: nexus-auth
Username: admin
Password: <nexus-password>
```

#### **Issue 5: "Shared library not found"**
```
Error: Library roboshop-shared-library not found
```

**Solution:**
1. Configure library in Jenkins global settings
2. Check repository URL is correct
3. Verify branch name (main/master)

#### **Issue 6: "Build timeout"**
```
Error: Build timed out (after 60 minutes)
```

**Solution:**
```groovy
options {
    timeout(time: 2, unit: 'HOURS')  // Increase timeout
}
```

#### **Issue 7: "Out of disk space"**
```
Error: No space left on device
```

**Solution:**
```groovy
post {
    always {
        cleanWs()  // Clean workspace after build
        sh 'docker system prune -af'  // Clean Docker
    }
}
```

#### **Issue 8: "Parallel stages fail randomly"**
```
Error: Stage failed due to resource contention
```

**Solution:**
```groovy
options {
    disableConcurrentBuilds()  // Prevent parallel builds
}
```

Or use resource locking:
```groovy
lock(resource: 'database') {
    // Stages that need exclusive access
}
```

---

## Advanced Topics

### **1. Dynamic Stages**

```groovy
def environments = ['dev', 'qa', 'staging', 'prod']

stages {
    script {
        environments.each { env ->
            stage("Deploy to ${env}") {
                sh "deploy.sh ${env}"
            }
        }
    }
}
```

### **2. Matrix Builds**

```groovy
matrix {
    axes {
        axis {
            name 'BROWSER'
            values 'chrome', 'firefox', 'safari'
        }
        axis {
            name 'OS'
            values 'linux', 'windows', 'mac'
        }
    }
    stages {
        stage('Test') {
            steps {
                echo "Testing on ${BROWSER} - ${OS}"
            }
        }
    }
}
```

### **3. Kubernetes Agent**

```groovy
pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: maven
    image: maven:3.8-jdk-11
    command: ['cat']
    tty: true
  - name: docker
    image: docker:latest
    command: ['cat']
    tty: true
'''
        }
    }
    stages {
        stage('Build') {
            steps {
                container('maven') {
                    sh 'mvn clean package'
                }
            }
        }
        stage('Docker Build') {
            steps {
                container('docker') {
                    sh 'docker build -t myapp .'
                }
            }
        }
    }
}
```

### **4. GitOps Integration**

```groovy
stage('Update GitOps Repo') {
    steps {
        script {
            sh """
                git clone https://github.com/user/gitops-repo.git
                cd gitops-repo
                sed -i 's/image:.*/image: myapp:${packageVersion}/' deployment.yaml
                git add .
                git commit -m 'Update image to ${packageVersion}'
                git push
            """
        }
    }
}
```

---

## Key Takeaways

### **Infrastructure Pipeline (Jenkinsfile-vpc01)**
1. Automates VPC provisioning with Terraform
2. Runs on demand or scheduled
3. Needs improvement: Add plan stage, input approval, better error handling

### **Shared Library Pattern**
1. **Router** (`pipelineDecision.groovy`): Decides which pipeline to run
2. **Implementations** (`nodeJSVMCI.groovy`, `javaVMCI.groovy`): Specific CI/CD logic
3. **Benefits**: Code reuse, consistency, centralized updates

### **Node.js CI Pipeline**
1. Version extraction from package.json
2. Build → Test → Scan → Package → Publish
3. Automatic deployment to DEV
4. Integration with Nexus for artifact storage

### **CI/CD Flow**
```
Code Push → Jenkins Webhook → Shared Library → Build & Test → 
Nexus Upload → Deployment Trigger → Environment Deployment → Health Check
```

### **Best Practices Summary**
1. Use shared libraries for reusability
2. Never hardcode credentials
3. Always include testing stages
4. Implement proper error handling
5. Use artifacts repositories (Nexus)
6. Add manual approval for production
7. Clean up workspace after builds
8. Send notifications on build status

---

## Quick Reference

### **Jenkins Pipeline Syntax**

```groovy
// Define pipeline
pipeline {
    agent { label 'agent-name' }
    
    environment {
        VAR = 'value'
    }
    
    parameters {
        string(name: 'PARAM', defaultValue: 'value')
    }
    
    stages {
        stage('Stage Name') {
            steps {
                sh 'command'
            }
        }
    }
    
    post {
        always { echo 'Always runs' }
        success { echo 'On success' }
        failure { echo 'On failure' }
    }
}
```

### **Common Jenkins Commands**

```groovy
// Shell execution
sh 'ls -la'

// Script block
script {
    def var = 'value'
    echo "Variable: ${var}"
}

// Credentials
withCredentials([usernamePassword(credentialsId: 'id', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
    sh 'echo $USER'
}

// Trigger job
build job: 'job-name', parameters: [string(name: 'VERSION', value: '1.0')]

// File operations
readJSON(file: 'file.json')
writeFile(file: 'output.txt', text: 'content')
```

---

This comprehensive guide covers everything you need to understand, work with, and improve this Jenkins pipeline setup!
