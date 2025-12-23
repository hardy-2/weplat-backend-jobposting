pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: maven
    image: maven:3.6.3-jdk-11
    command:
    - cat
    tty: true
  - name: kaniko
    image: gcr.io/kaniko-project/executor:debug
    imagePullPolicy: Always
    command:
    - /busybox/cat
    tty: true
    volumeMounts:
      - name: jenkins-docker-cfg
        mountPath: /kaniko/.docker
  volumes:
  - name: jenkins-docker-cfg
    projected:
      sources:
      - secret:
          name: registry-credentials
          items:
            - key: .dockerconfigjson
              path: config.json
"""
        }
    }

    environment {
        DOCKER_IMAGE = "lch7087/jobposting"
        IMAGE_TAG = "v${env.BUILD_NUMBER}"
    }

    stages {
        stage('Git Clone from gitSCM') {
            steps {
                script {
                    try {
                        git branch: 'main',
                            credentialsId: 'Git',
                            url: 'https://github.com/hardy-2/weplat-backend-jobposting'
                        sh "ls -lat" 
                        sh "pwd"
                        env.cloneResult = true
                    } catch (error) {
                        error.printStackTrace()
                        env.cloneResult = false
                        currentBuild.result = 'FAILURE'
                    }
                }
            }
        }

        stage('Build JAR with Gradle') {
            steps {
                script {
                    try {
                        sh "chmod +x gradlew"
                        sh "./gradlew bootJar -x test -x integrationTest"
                        env.buildResult = true
                    } catch (error) {
                        error.printStackTrace()
                        env.buildResult = false
                        currentBuild.result = 'FAILURE'
                    }
                }
            }
        }

        stage('Check JAR File') {
            steps {
                script {
                    if (env.buildResult.toBoolean()) {
                        if (fileExists('build/libs/jobposting-0.0.1-SNAPSHOT.jar')) {
                            echo 'JAR file is present at build/libs/jobposting-0.0.1-SNAPSHOT.jar'
                        } else {
                            echo 'JAR file is not present'
                            currentBuild.result = 'FAILURE'
                        }
                    }
                }
            }
        }

        stage('Build and Push Docker Image') {
            steps {
                container('kaniko') {
                    script {
                        sh """
                        /kaniko/executor \
                        --context $WORKSPACE \
                        --dockerfile $WORKSPACE/Dockerfile \
                        --destination ${DOCKER_IMAGE}:${IMAGE_TAG} \
                        --cache=true
                        """
                    }
                }
            }
        }

        stage('Update Manifest & Push to GitHub') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'Github-id', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                        sh """
                        # Git 사용자 설정
                        git config user.email "Your Name"
                        git config user.name "your.email@example.com"

                        # YAML 파일에서 이미지 태그 수정
                        sed -i "s|image: ${DOCKER_IMAGE}:.*|image: ${DOCKER_IMAGE}:${IMAGE_TAG}|g" jobposting-deploy.yml

                        # 변경사항 확인
                        cat jobposting-deploy.yml | grep image

                        # Git Add, Commit & Push
                        git add jobposting-deploy.yml
                        git commit -m "Update image tag to ${IMAGE_TAG} [skip ci]"
                        git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/hardy-2/weplat-backend-jobposting.git main
                        """
                    }
                }
            }
        }
    }
}
