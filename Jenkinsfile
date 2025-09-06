pipeline {
    agent any

    environment {
        AWS_REGION = "ap-south-1"
        IMAGE_NAME = "java-app"
        ACCOUNT_ID = "739275464858"
        ECR_REPO   = "${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${IMAGE_NAME}"
        SONARQUBE   = "SonarCloud"  // Name configured in Jenkins SonarQube Servers
        SONAR_TOKEN = credentials('sonarcloud-token') // Jenkins credential ID
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/Gagan0826/boardgame'
            }
        }

        stage('Build JAR') {
            steps {
                sh """
                    docker run --rm \
                        -v ${WORKSPACE}:/app \
                        -w /app \
                        -v $HOME/.m2:/root/.m2 \
                        maven:3.9.6-eclipse-temurin-21 \
                        mvn clean package -DskipTests
                """
            }
        }
        
    stage('SonarCloud Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE}") {
                    sh """
                        docker run --rm \
                            -v ${WORKSPACE}:/app \
                            -w /app \
                            -v $HOME/.m2:/root/.m2 \
                            maven:3.9.6-eclipse-temurin-21 \
                            mvn verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
                                -Dsonar.projectKey=Gagan0826_boardgame \
                                -Dsonar.organization=gagan0826 \
                                -Dsonar.host.url=https://sonarcloud.io \
                                -Dsonar.login=${SONAR_TOKEN}
                    """
                }
            }
        }

        // stage('Push JAR to Nexus') {
        //     steps {
        //         withCredentials([usernamePassword(credentialsId: 'nexus-cred', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASSWORD')]) {
        //             script {
        //                 def version = sh(returnStdout: true, script: """
        //                     docker run --rm \
        //                         -v ${WORKSPACE}:/app \
        //                         -w /app \
        //                         maven:3.9.6-eclipse-temurin-21 \
        //                         mvn help:evaluate -Dexpression=project.version -q -DforceStdout
        //                 """).trim()

        //                 def (repoUrl, repoId) = version.endsWith("SNAPSHOT") 
        //                     ? ["http://13.201.118.87:8081/repository/maven-snapshots/", "maven-snapshots"]
        //                     : ["http://13.201.118.87:8081/repository/maven-releases/", "maven-releases"]

        //                 sh """
        //                      docker run --rm \
        //                         -e NEXUS_USER=$NEXUS_USER \
        //                         -e NEXUS_PASSWORD=$NEXUS_PASSWORD \
        //                         -v ${WORKSPACE}:/app \
        //                         -w /app \
        //                         -v $HOME/.m2:/root/.m2 \
        //                         maven:3.9.6-eclipse-temurin-21 \
        //                         mvn deploy:deploy-file \
        //                             -DgroupId=com.javaproject \
        //                             -DartifactId=database_service_project \
        //                             -Dversion=${version} \
        //                             -Dpackaging=jar \
        //                             -Dfile=target/database_service_project-${version}.jar \
        //                             -Durl=${repoUrl} \
        //                             -DrepositoryId=${repoId} \
        //                             -Dusername=$NEXUS_USER \
        //                             -Dpassword=$NEXUS_PASSWORD
        //                 """
        //             }
        //         }
        //     }
        // }

stage('Docker Build & Push to ECR') {
    steps {
        script {
            def buildTag = "${BUILD_NUMBER}"   // only the number

            sh """
                # Build image with only the build number
                docker build -t ${ECR_REPO}:${buildTag} .

                # Login to ECR
                aws ecr get-login-password --region ${AWS_REGION} \
                    | docker login --username AWS --password-stdin ${ECR_REPO}

                # Push build number tag
                docker push ${ECR_REPO}:${buildTag}

                # Tag the same image as latest (no rebuild)
                docker tag ${ECR_REPO}:${buildTag} ${ECR_REPO}
            """
               }
            }
        }

    stage('Deploy to EKS') {
        steps {
            script {
            sh """
                aws eks update-kubeconfig --region ${AWS_REGION} --name Java-application-deployment
                sed -i "s|<tag>|${BUILD_NUMBER}|g" deployment-service.yaml
                kubectl apply -f deployment-service.yaml
            """
                }
            }
        }
    }
}
