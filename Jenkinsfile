pipeline {
    agent any

    tools {
        jdk "jdk17"
        maven "M3"
    }
    
    stages {
        stage('Git clone') {
            steps {
                echo 'Git clone'
                git url: 'https://github.com/sjh4616/spring-petclinic.git',
                    branch: 'efficient-webjars'
            }
            post {
                success {
                    echo 'Git Clone Success!!'
                }
                failure {
                    echo 'Git Clone Fail'
                }
            }
        }
    
        stage('Maven Build') {
            steps {
                echo 'Maven Build'
                sh 'mvn -Dmaven.test.failure.ignore=true clean package'
            }        
            post {
                success {
                    junit 'target/surefire-reports/**/*.xml'
                }
            }
        }
    }    
}
