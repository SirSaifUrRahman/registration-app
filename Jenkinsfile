pipeline {
    agent {
        label 'habib-node'
    }

    tools {
        maven 'Maven3'
        jdk 'Java17'
    }

    environment {
        SONAR_HOME = tool "Sonar"
        GIT_REPO = 'https://github.com/SirSaifUrRahman/registration-app.git'

        APP_NAME = "register-app-pipeline"
        RELEASE = "1.0.0"
        DOCKER_USER = "saif764"
        DOCKERHUB_LABEL = "dockerhub-cred"
        IMAGE_NAME = "${DOCKER_USER}/${APP_NAME}"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
        JENKINS_API_TOKEN = credentials("JENKINS_API_TOKEN")
        BUILD_NUM_OF_CI = "${env.BUILD_NUMBER}"
    }

    parameters {
        string(name: 'BRANCH_NAME', defaultValue: 'main', description: 'Git branch to build')
        //booleanParam(name: 'RUN_TESTS', defaultValue: true, description: 'Run tests after building the project')
    }
    
    stages {
        stage('Cleanup Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Clone Repository') {
            steps {
                echo "Cloning ${GIT_REPO} branch: ${params.BRANCH_NAME}"
                git branch: "${params.BRANCH_NAME}", url: "${GIT_REPO}", credentialsId: 'github'
                sh "echo Printing Build Number of Job"
                sh "echo Build Number of CI Job is: ${BUILD_NUM_OF_CI}"
            }
        }

        stage('Build Application') {
            steps {
                withEnv(["JAVA_HOME=${tool('Java17')}", 'PATH+JAVA=${tool("Java17")}/bin']) {
                    sh 'echo "JAVA_HOME=$JAVA_HOME"'
                    sh "${tool('Maven3')}/bin/mvn clean package"
                }
            }
        }

        stage('Test') {
            // when {
            //     expression { return params.RUN_TESTS }
            // }
            steps {
                withEnv(["JAVA_HOME=${tool('Java17')}", 'PATH+JAVA=${tool("Java17")}/bin']) {
                    sh 'echo "JAVA_HOME=$JAVA_HOME"'
                    sh "${tool('Maven3')}/bin/mvn test"
                }
            }
        }

        stage("SonarQube: Code Analysis"){
            steps{
                script{
                    withSonarQubeEnv(credentialsId: 'sonar-token'){
                        sh "mvn sonar:sonar"
                    }
                }
            }
        }
        
        stage("SonarQube: Code Quality Gates"){
            steps{
                script{
                     waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }

        // stage("Build & Push Docker Image"){
        //     steps {
        //         script {
        //             docker.withRegistry('', DOCKER_PASS) {
        //             // Build Docker image with specific tag
        //             docker_image = docker.build("${IMAGE_NAME}:${IMAGE_TAG}")
        //             }
                

        //             docker.withRegistry('', DOCKER_PASS) {
        //             // Push specific version tag
        //             docker_image.push("${IMAGE_TAG}")
            
        //             // Push latest tag
        //             docker_image.push("latest")
        //             }
        //         }
        //     }
  
        // }
        stage("Build & Push Docker Image") {
            steps {
                script {
                    // Securely inject DockerHub credentials (username/password)
                    withCredentials([usernamePassword(
                        credentialsId: "${DOCKERHUB_LABEL}", // Update this ID
                        usernameVariable: "DOCKER_USER",
                        passwordVariable: "DOCKER_PASS"
                    )]) {
                        def imageNameWithTag = "${IMAGE_NAME}:${IMAGE_TAG}"

                        // Build Docker image
                        def dockerImage = docker.build(imageNameWithTag)

                        // Push using Docker registry with credentials
                        docker.withRegistry('https://index.docker.io/v1/', "${DOCKERHUB_LABEL}") {
                            dockerImage.push("${IMAGE_TAG}")   // Push versioned tag
                            dockerImage.push("latest")         // Push latest tag
                        }

                        echo "Docker image ${imageNameWithTag} built and pushed successfully."
                    }
                }
            }
        }


        stage("Trivy Scan") {
            steps {
                script {
                    sh '''
                        docker run \
                        -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy image saif764/register-app-pipeline:latest \
                        --no-progress --scanners vuln --exit-code 0 \
                        --severity HIGH,CRITICAL --format table
                    '''
                }
            }
        }

        
        stage("Clean Up Artifacts"){
            steps{
                script{
                    sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker rmi ${IMAGE_NAME}:latest"
                }
            }
        }


        stage("Trigger CD Pipeline") {
            steps {
                script {

                    withCredentials([string(credentialsId: 'JENKINS_API_TOKEN', variable: 'TOKEN')]) {
                    sh """
                        curl -v -k --user saif:$TOKEN \\
                        -X POST \\
                        -H "cache-control: no-cache" \\
                        -H "content-type: application/x-www-form-urlencoded" \\
                        --data 'BUILD_NUM_OF_CD=${BUILD_NUM_OF_CI}' \\
                        --data 'APP_NAME=${APP_NAME}' \\
                        --data 'RELEASE=${RELEASE}' \\
                        --data 'DOCKER_USER=${DOCKER_USER}' \\
                        --data 'IMAGE_NAME=${IMAGE_NAME}' \\
                        --data 'IMAGE_TAG=${IMAGE_TAG}' \\
                        "http://192.168.43.75:8080/job/gitops-register-app-cd/buildWithParameters?token=gitops-token"
                    """
                    }

                    // sh """
                    //     curl -v -k --user saif:${JENKINS_API_TOKEN} \\
                    //     -X POST \\
                    //     -H 'cache-control: no-cache' \\
                    //     -H 'content-type: application/x-www-form-urlencoded' \\
                    //     --data 'APP_NAME=${APP_NAME}' \\
                    //     --data 'RELEASE=${RELEASE}' \\
                    //     --data 'DOCKER_USER=${DOCKER_USER}' \\
                    //     --data 'IMAGE_NAME=${IMAGE_NAME}' \\
                    //     --data 'IMAGE_TAG=${IMAGE_TAG}' \\
                    //     'http://192.168.43.75:8080/job/gitops-register-app-cd/buildWithParameters?token=gitops-token'
                    // """
                }
            }
        }

    }
}
