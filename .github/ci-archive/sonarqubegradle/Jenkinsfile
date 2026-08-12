<![CDATA[
// Source: https://github.com/jenkinsci/pipeline-examples (MIT License)
// Test case: SonarQube analysis with quality gates
node {
    stage('Checkout') {
        checkout scm
    }

    stage('Build') {
        sh './gradlew clean build'
    }

    stage('Test') {
        sh './gradlew test'
        publishTestResults testResultsPattern: 'build/test-results/test/*.xml'
    }

    stage('SonarQube Analysis') {
        withSonarQubeEnv('SonarQube') {
            sh './gradlew sonarqube \
                -Dsonar.projectKey=my-project \
                -Dsonar.projectName="My Project" \
                -Dsonar.projectVersion=${BUILD_NUMBER} \
                -Dsonar.sources=src/main \
                -Dsonar.tests=src/test \
                -Dsonar.java.binaries=build/classes \
                -Dsonar.coverage.jacoco.xmlReportPaths=build/reports/jacoco/test/jacocoTestReport.xml'
        }
    }

    stage("Quality Gate") {
        timeout(time: 10, unit: 'MINUTES') {
            def qg = waitForQualityGate()
            if (qg.status != 'OK') {
                error "Pipeline aborted due to quality gate failure: ${qg.status}"
            }
        }
    }

    stage('Package') {
        sh './gradlew bootJar'
        archiveArtifacts artifacts: 'build/libs/*.jar', fingerprint: true
    }

    stage('Docker Build') {
        script {
            def image = docker.build("my-app:${BUILD_NUMBER}")
            docker.withRegistry('https://registry.example.com', 'docker-registry-credentials') {
                image.push()
                image.push("latest")
            }
        }
    }
}
    ]]>