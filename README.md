# CI/CD Pipeline with Nexus, Sonarqube and Jenkins

This document describes the Continuous Integration and Continuous Deployment (CI/CD) pipeline for the vProfile project using Jenkins, Maven, SonarQube, and Nexus Repository Manager.

## Overview

The pipeline automates the process of fetching the code, building the project, running tests, performing static code analysis, and publishing the artifacts to Nexus. Notifications and quality gates are also integrated to ensure code quality and seamless deployment.
![arsitektur 07 devops project](https://github.com/user-attachments/assets/ecef2965-32d9-42a8-8d95-15e325363a0e)

## Tools and Technologies Used
- **Jenkins**: Orchestrates the CI/CD process.
- **Maven**: Manages project builds, dependencies, and testing.
- **SonarQube**: Provides static code analysis and quality gates.
- **Nexus Repository**: Stores versioned artifacts for deployment.
- **Git**: SCM system for version control.

## Pipeline Stages

### 1. Fetch Code
The pipeline begins by fetching the code from the `paac` branch of the `vprofile-project` Git repository.

```groovy
stage('Fetch Code') {
    steps {
        git branch: 'paac', url: 'https://github.com/devopshydclub/vprofile-project.git'
    }
}
```

### 2. Build the Project (Skip Tests)
In this stage, the Maven build is performed, skipping the unit tests for faster build times.

```groovy
stage('BUILD') {
    steps {
        sh 'mvn clean install -DskipTests'
    }
    post {
        success {
            archiveArtifacts artifacts: '**/target/*.war'
        }
    }
}
```

### 3. Unit Tests
Maven runs the unit tests to validate the code at the unit level.

```groovy
stage('UNIT TEST') {
    steps {
        sh 'mvn test'
    }
}
```

### 4. Integration Tests
Integration tests are executed using Maven, while skipping unit tests for this phase.

```groovy
stage('INTEGRATION TEST') {
    steps {
        sh 'mvn verify -DskipUnitTests'
    }
}
```

### 5. Code Analysis with Maven Checkstyle
Maven Checkstyle is used to perform static code analysis based on predefined coding standards.

```groovy
stage('CODE ANALYSIS WITH CHECKSTYLE') {
    steps {
        sh 'mvn checkstyle:checkstyle'
    }
    post {
        success {
            echo 'Generated Analysis Result'
        }
    }
}
```

### 6. Code Analysis with SonarQube
SonarQube is integrated to analyze the code quality, providing detailed metrics and enforcing quality gates.

- **SonarQube Scanner**: Runs the analysis.
- **Quality Gate**: The pipeline will halt if the code fails the quality gate.

```groovy
stage('CODE ANALYSIS with SONARQUBE') {
    environment {
        scannerHome = tool 'mysonarscanner4'
    }

    steps {
        withSonarQubeEnv('sonar-pro') {
            sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
               -Dsonar.projectName=vprofile-repo \
               -Dsonar.projectVersion=1.0 \
               -Dsonar.sources=src/ \
               -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
               -Dsonar.junit.reportsPath=target/surefire-reports/ \
               -Dsonar.jacoco.reportsPath=target/jacoco.exec \
               -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
        }

        timeout(time: 10, unit: 'MINUTES') {
            waitForQualityGate abortPipeline: true
        }
    }
}
```

### 7. Publish Artifacts to Nexus Repository Manager
After the successful build and quality checks, the pipeline uploads the generated artifacts (WAR file and POM) to Nexus Repository Manager.

**Environment Variables**:
- `NEXUS_VERSION`, `NEXUS_PROTOCOL`, `NEXUS_URL`: Defines the Nexus configuration.
- `NEXUS_REPOSITORY`, `NEXUS_REPO_ID`, `NEXUS_CREDENTIAL_ID`: Nexus repository and credentials configuration.
- `ARTVERSION`: Sets the artifact version dynamically using the build ID.

```groovy
stage("Publish to Nexus Repository Manager") {
    steps {
        script {
            pom = readMavenPom file: "pom.xml";
            filesByGlob = findFiles(glob: "target/*.${pom.packaging}");
            artifactPath = filesByGlob[0].path;
            artifactExists = fileExists artifactPath;
            if(artifactExists) {
                nexusArtifactUploader(
                        nexusVersion: NEXUS_VERSION,
                        protocol: NEXUS_PROTOCOL,
                        nexusUrl: NEXUS_URL,
                        groupId: pom.groupId,
                        version: ARTVERSION,
                        repository: NEXUS_REPOSITORY,
                        credentialsId: NEXUS_CREDENTIAL_ID,
                        artifacts: [
                                [artifactId: pom.artifactId,
                                 classifier: '',
                                 file: artifactPath,
                                 type: pom.packaging],
                                [artifactId: pom.artifactId,
                                 classifier: '',
                                 file: "pom.xml",
                                 type: "pom"]
                        ]
                );
            }
            else {
                error "*** File: ${artifactPath}, could not be found";
            }
        }
    }
}
```

## Environment Variables
The following environment variables are set to streamline the Nexus artifact upload and SonarQube analysis:

- **NEXUS_VERSION**: The version of Nexus (e.g., `nexus3`).
- **NEXUS_PROTOCOL**: The communication protocol (e.g., `http`).
- **NEXUS_URL**: The IP and port of the Nexus server.
- **NEXUS_REPOSITORY**: The name of the Nexus repository (e.g., `vprofile-release`).
- **NEXUS_REPO_ID**: The repository ID for the project.
- **NEXUS_CREDENTIAL_ID**: The credential ID used to authenticate with Nexus.
- **ARTVERSION**: The artifact version, dynamically set using Jenkins' `BUILD_ID`.

## Conclusion
This CI/CD pipeline ensures seamless integration and deployment of the vProfile project. By integrating Maven, SonarQube, and Nexus Repository Manager, it provides automatic code quality checks, artifact versioning, and deployment to a centralized artifact repository. It is crucial to configure Jenkins properly with the correct credentials, environment variables, and plugins for Maven, SonarQube, and Nexus.

Ensure the following configurations:
- Maven Checkstyle for static code checks.
- SonarQube integration for comprehensive code analysis.
- Webhooks and quality gates for early detection of code issues.
- Nexus Repository Manager for artifact version control and storage.
![pipeline output](https://github.com/user-attachments/assets/797896ee-13c7-40c1-a1f1-9e29056d580c)
