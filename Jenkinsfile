node {
    // CI 단계 시작
    // Checkout SCM 단계: 소스 코드를 SCM에서 체크아웃합니다.
    stage('Checkout SCM') {
        deleteDir() // 작업 디렉토리를 삭제하여 깨끗한 상태로 시작합니다.
        checkout scm // 소스 코드를 체크아웃합니다.
    }

    // Docker Build 단계: Docker 이미지를 빌드합니다.
    stage('Docker Build') {
        sh 'docker build -t 110.15.58.113:8083/repository/app-test:latest .' // Docker 이미지를 Nexus 레지스트리에 태그를 붙여 빌드합니다.
    }

    // Docker Push 단계: 빌드된 Docker 이미지를 Nexus 레지스트리에 푸시합니다.
    stage('Docker Push') {
        withCredentials([usernamePassword(credentialsId: 'nexus-credentials', usernameVariable: 'NEXUS_USERNAME', passwordVariable: 'NEXUS_PASSWORD')]) {
            // Nexus 레지스트리에 로그인합니다.
            sh 'echo $NEXUS_PASSWORD | docker login -u $NEXUS_USERNAME --password-stdin http://110.15.58.113:8083'
            // 빌드된 Docker 이미지를 Nexus 레지스트리에 푸시합니다.
            sh 'docker push 110.15.58.113:8083/repository/app-test:latest'
        }
    }

    // Tag Version Up 단계: Git 태그를 증가시키고 푸시합니다.
    stage('Tag Version Up') {
        script {
            withCredentials([usernamePassword(credentialsId: 'github-credentials', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                // Git 사용자 이름과 이메일을 전역 설정으로 설정
                sh 'git config --global user.name "${GIT_USERNAME}"'
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
    // CI 단계 종료


    // CD 단계 시작
    // Docker Pull Images 단계: Docker 이미지를 풀링합니다.
    stage('Docker Pull Images') {
        withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
            sh 'echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin'
            sh "docker pull 110.15.58.113:8083/repository/app-test:latest" // 실제 백엔드 이미지 이름으로 변경
            sh 'docker logout'
        }
    }

    // Deploy Stack 단계: Docker Stack을 배포합니다.
    stage('Deploy Stack') {
        sh "docker stack deploy --compose-file docker-compose.yaml jenkins_flask_app"
    }
    // CD 단계 종료
}
