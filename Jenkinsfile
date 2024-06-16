pipeline {
    agent any

    environment {
        SLACK_CHANNEL = '#99_알림'
        SLACK_COLOR_SUCCESS = '#00FF00'
        SLACK_COLOR_FAILURE = '#FF0000'
        SLACK_COLOR_STARTED = '#FFFF00'
    }

    stages {
        stage('Checkout SCM') {
            steps {
                deleteDir() // 작업 디렉토리를 삭제하여 깨끗한 상태로 시작합니다.
                checkout scm // 소스 코드를 체크아웃합니다.
                slackSend (channel: SLACK_CHANNEL, color: SLACK_COLOR_STARTED, message: "STARTED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
            }
        }

        stage('Docker Build') {
            steps {
                sh 'docker build -t 110.15.58.113:8083/repository/app-test:latest .' // Docker 이미지를 Nexus 레지스트리에 태그를 붙여 빌드합니다.
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'nexus-credentials', usernameVariable: 'NEXUS_USERNAME', passwordVariable: 'NEXUS_PASSWORD')]) {
                    // Nexus 레지스트리에 로그인합니다.
                    sh 'echo $NEXUS_PASSWORD | docker login -u $NEXUS_USERNAME --password-stdin http://110.15.58.113:8083'
                    // 빌드된 Docker 이미지를 Nexus 레지스트리에 푸시합니다.
                    sh 'docker push 110.15.58.113:8083/repository/app-test:latest'
                }
            }
        }

        stage('Tag Version Up') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'github-credentials', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                        // Git 사용자 이름과 이메일을 전역 설정으로 설정
                        sh 'git config --global user.name "$GIT_USERNAME"'
                        sh 'git config --global user.email "donghyun4591@gmail.com"'

                        // 최신 태그를 가져와 버전을 증가시킵니다. 태그가 없으면 기본 태그로 0.0.0을 사용합니다.
                        def version = sh(script: "git describe --tags --abbrev=0 || echo '0.0.0'", returnStdout: true).trim()
                        def (major, minor, patch) = version.tokenize('.').collect { it.toInteger() }
                        patch += 1 // 패치 번호를 증가시킵니다.
                        def newVersion = "${major}.${minor}.${patch}"

                        // 새로운 태그를 생성하고 푸시합니다.
                        sh "git tag -a ${newVersion} -m 'Version ${newVersion}'"

                        // 브랜치 명시적으로 체크아웃
                        sh 'git checkout master' // 여기서 main 또는 원격 브랜치 이름으로 변경

                        // 푸시할 때 브랜치 이름 명시
                        sh "git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/dongkoony/BootGenie.git HEAD:master"
                        sh "git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/dongkoony/BootGenie.git ${newVersion}"
                    }
                }
            }
        }

        stage('Docker Pull Images') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                    sh 'echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin'
                    sh "docker pull 110.15.58.113:8083/repository/app-test:latest" // 실제 백엔드 이미지 이름으로 변경
                    sh 'docker logout'
                }
            }
        }

        stage('Deploy Stack') {
            steps {
                // ensure docker-compose.yml file exists
                script {
                    def composeFile = 'docker-compose.yml'
                    if (!fileExists(composeFile)) {
                        error "File ${composeFile} not found"
                    }
                }
                sh "docker stack deploy --compose-file docker-compose.yaml my_flask_app"
            }
        }
    }

    post {
        success {
            slackSend (channel: SLACK_CHANNEL, color: SLACK_COLOR_SUCCESS, message: "SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
        }
        failure {
            slackSend (channel: SLACK_CHANNEL, color: SLACK_COLOR_FAILURE, message: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
        }
    }
}
