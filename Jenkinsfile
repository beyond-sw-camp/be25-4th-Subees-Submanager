/*
* 파이프라인 실행환경 세팅
* Kubernetes 위에 임시 Jenkins agent Pod를 띄워서 실행
* 파드 안에 컨테이너 2개
*/
pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
metadata:
  name: subees-app-agent
spec:
  containers:
  - name: maven
    image: maven:3.9.9-eclipse-temurin-21-alpine
    command:
    - cat
    tty: true
  - name: docker
    image: docker:28.5.1-cli-alpine3.22
    command:
    - cat
    tty: true
    volumeMounts:
    - mountPath: "/var/run/docker.sock"
      name: docker-socket
  volumes:
  - name: docker-socket
    hostPath:
      path: "/var/run/docker.sock"
'''
        }
    }
    options {
        skipDefaultCheckout(true)
    }

// 환경 변수
    environment {
        FRONT_IMAGE = 'myang12/subees-frontend'
        BACK_IMAGE = 'myang12/subees-backend'
        DOCKER_CREDENTIALS_ID = 'dockerhub-access'
        DISCORD_WEBHOOK_CREDENTIALS_ID = 'discord-webhook'
    }
/* 
* 첫 스테이지 : 변경 감지
* 현재 커밋과 바로 이전 커밋을 비교하여 바뀐 파일 목록을 가져옴
* 백엔드, 프론트 아래 파일이 하나라도 바뀌면 빌드
* k8s 아래 수정 시 도커로그인, 백, 프론트 빌드, 트리거 스킵
*/
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Detect Changes') {
            steps {
                script {
                    def changedText = sh(
                        script: 'git diff --name-only HEAD~1',
                        returnStdout: true
                    ).trim()

                    def changedFiles = changedText ? changedText.split("\\n") : []

                    echo "Changed files:\n${changedFiles.join('\n')}"

                    env.SHOULD_BUILD_FRONT = changedFiles.any {
                        it.startsWith("fronted/")
                    } ? "true" : "false"

                    env.SHOULD_BUILD_BACK = changedFiles.any {
                        it.startsWith("backend/")
                    } ? "true" : "false"

                    env.ONLY_K8S_CHANGED = changedFiles.size() > 0 && changedFiles.every {
                        it.startsWith("k8s/")
                    } ? "true" : "false"

                    echo "SHOULD_BUILD_FRONT : ${env.SHOULD_BUILD_FRONT}"
                    echo "SHOULD_BUILD_BACK : ${env.SHOULD_BUILD_BACK}"
                    echo "ONLY_K8S_CHANGED : ${env.ONLY_K8S_CHANGED}"

                    if (env.ONLY_K8S_CHANGED == "true") {
                        echo "Only k8s manifest changed. Docker image build will be skipped."
                    }
                }
            }
        }

/*
* 프론트, 백엔드 중 하나라도 빌드할 때만 도커 허브 로그인
*/
        stage('Docker Login') {
            when {
                expression {
                    return env.SHOULD_BUILD_FRONT == "true" || env.SHOULD_BUILD_BACK == "true"
                }
            }

            steps {
                container('docker') {
                    withCredentials([usernamePassword(
                        credentialsId: DOCKER_CREDENTIALS_ID,
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )]) {
                        sh 'echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin'
                    }
                }
            }
        }
/*
* backend/ 변경 있을 시 실행
* Maven 컨테이너에서 Spring Boot jar 파일 만든다. 테스트 실행 X
* 도커 컨테이너에서  Jenkins BUILD_NUMBER를 Docker image tag로 사용하고 도커 허브 푸쉬
*/
        stage('Backend Image Build & Push') {
            when {
                expression {
                    return env.SHOULD_BUILD_BACK == "true"
                }
            }

            steps {
                container('maven') {
                    dir('backend/subscription') {
                        sh 'mvn -B clean package -DskipTests'
                    }
                }

                container('docker') {
                    dir('backend/subscription') {
                        script {
                            def buildNumber = "${env.BUILD_NUMBER}"

                            withEnv(["DOCKER_IMAGE_VERSION=${buildNumber}"]) {
                                sh 'docker -v'
                                sh 'echo $BACK_IMAGE:$DOCKER_IMAGE_VERSION'
                                sh 'docker build -t $BACK_IMAGE:$DOCKER_IMAGE_VERSION ./'
                                sh 'docker image inspect $BACK_IMAGE:$DOCKER_IMAGE_VERSION'
                                sh 'docker push $BACK_IMAGE:$DOCKER_IMAGE_VERSION'
                            }
                        }
                    }
                }
            }
        }
/*
* fronted/ 변경이 있을 때만 실행
* 프론트 도커파일 내부에서 npm ci, npm run build Nginx 이미지 생성까지 처리
*/
        stage('Frontend Image Build & Push') {
            when {
                expression {
                    return env.SHOULD_BUILD_FRONT == "true"
                }
            }

            steps {
                container('docker') {
                    dir('fronted') {
                        script {
                            def buildNumber = "${env.BUILD_NUMBER}"

                            withEnv(["DOCKER_IMAGE_VERSION=${buildNumber}"]) {
                                sh 'docker -v'
                                sh 'echo $FRONT_IMAGE:$DOCKER_IMAGE_VERSION'
                                sh 'docker build -t $FRONT_IMAGE:$DOCKER_IMAGE_VERSION ./'
                                sh 'docker image inspect $FRONT_IMAGE:$DOCKER_IMAGE_VERSION'
                                sh 'docker push $FRONT_IMAGE:$DOCKER_IMAGE_VERSION'
                            }
                        }
                    }
                }
            }
        }
/*
* 이미지를 새로 만들었을 시 두번 째 jenkins job 호출
* 이번에 만든 이미지 태그, 프론트, 백 빌드 여부 k8s job 끝날 때까지 첫 job wait
*/
        stage('Trigger k8s-manifests Job') {
            when {
                expression {
                    return env.SHOULD_BUILD_FRONT == "true" || env.SHOULD_BUILD_BACK == "true"
                }
            }

            steps {
                script {
                    def buildNumber = "${env.BUILD_NUMBER}"

                    build job: 'subees-k8s-manifests',
                        parameters: [
                            string(name: 'DOCKER_IMAGE_VERSION', value: "${buildNumber}"),
                            string(name: 'DID_BUILD_FRONT', value: "${env.SHOULD_BUILD_FRONT}"),
                            string(name: 'DID_BUILD_BACK', value: "${env.SHOULD_BUILD_BACK}")
                        ],
                        wait: true
                }
            }
        }
    }
// 빌드 성공 여부 디코 알림
    post {
        success {
            withCredentials([string(
                credentialsId: DISCORD_WEBHOOK_CREDENTIALS_ID,
                variable: 'DISCORD_WEBHOOK_URL'
            )]) {
                sh '''
                    curl --max-time 10 --connect-timeout 5 \
                      -H "Content-Type: application/json" \
                      -d "{\\"content\\":\\"✅ Subees App Build 성공 - Build #$BUILD_NUMBER | Backend: $SHOULD_BUILD_BACK | Frontend: $SHOULD_BUILD_FRONT | Image Tag: $BUILD_NUMBER\\"}" \
                      "$DISCORD_WEBHOOK_URL" || echo "Discord notification failed."
                '''
            }
        }

        failure {
            withCredentials([string(
                credentialsId: DISCORD_WEBHOOK_CREDENTIALS_ID,
                variable: 'DISCORD_WEBHOOK_URL'
            )]) {
                sh '''
                    curl --max-time 10 --connect-timeout 5 \
                      -H "Content-Type: application/json" \
                      -d "{\\"content\\":\\"❌ Subees App Build 실패 - Build #$BUILD_NUMBER\\"}" \
                      "$DISCORD_WEBHOOK_URL" || echo "Discord notification failed."
                '''
            }
        }
    }
}
