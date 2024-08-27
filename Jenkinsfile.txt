pipeline {
    agent any

    environment {
        // Docker Hub credentials - Jenkins에서 설정한 자격 증명 ID 사용
        DOCKER_CREDENTIALS_ID = 'dockerhub'
        // Docker Hub username and image name
        DOCKER_IMAGE_NAME = 'jwlee2357/graduationwork'
        // Git branch to build
        BRANCH_NAME = 'main'
    }

    stages {
        stage('Checkout') {
            steps {
                // GitHub 리포지토리에서 코드 체크아웃
                git branch: "${BRANCH_NAME}", url: 'https://github.com/jwlee2357/graduationWork.git'
            }
        }

        stage('Set Execute Permission') {
            steps {
                // gradlew에 실행 권한 부여
                script {
                    sh 'chmod +x gradlew'
                }
            }
        }

        stage('Build JAR') {
            steps {
                // JAR 파일을 빌드
                script {
                    sh './gradlew build -x test'  // Gradle을 사용하는 경우
                    // 또는
                    // sh 'mvn package'  // Maven을 사용하는 경우
                }
            }
        }

        stage('Prepare Docker Build Context') {
            steps {
                // Docker 빌드 컨텍스트로 JAR 파일을 복사
                script {
                    sh 'cp build/libs/server-0.0.1-SNAPSHOT.jar .'
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                // Docker 이미지를 빌드
                script {
                    dockerImage = docker.build("${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}")
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', "${DOCKER_CREDENTIALS_ID}") {
                        dockerImage.push("${env.BUILD_NUMBER}")
                        dockerImage.push("latest")
                    }
                }
            }
        }

        stage('GitHub Clone Yaml Repository') {
            steps {
                echo 'Cloning GitHub Repository'
                git branch: 'main', credentialsId: 'github', url: 'https://github.com/jwlee2357/graduationWorkYAML.git'
            }
        }
        
        stage('Update deploy.yaml') {
            steps {
                script {
                    IMAGE_TAG = "${currentBuild.number}"  // Jenkins 빌드 번호를 이미지 태그로 사용
                    echo "Image tag: ${IMAGE_TAG}"
                    // sed 명령어를 사용하여 YAML 파일의 이미지를 업데이트
                    sh """
                        sed -i 's|image: .*|image: ${DOCKER_IMAGE_NAME}:${IMAGE_TAG}|' insurance-deployment.yaml
                    """
                }
            }
        }
        
        stage("Push to Git Repository") {
            steps {
                withCredentials([usernamePassword(credentialsId: 'github', usernameVariable: 'GITHUB_USER', passwordVariable: 'GITHUB_TOKEN')]) {
                    sh """
                        git config user.email "rink2357@naver.com"
                        git config user.name "${GITHUB_USER}"
                        git add .
                        git commit -m "Update image tag in insurance-deployment.yaml"
                        git push https://${GITHUB_USER}:${GITHUB_TOKEN}@github.com/jwlee2357/graduationWorkYAML.git HEAD:main
                    """
                }
            }
        }

        stage('Clean Up') {
            steps {
                // 로컬 Docker 이미지를 제거하여 공간을 확보
                sh "docker rmi ${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER} || true"
            }
        }
    }

    post {
        always {
            // 빌드 후 항상 수행되는 작업
            cleanWs() // 워크스페이스 정리
        }
    }
}