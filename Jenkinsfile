pipeline {
    agent any
    tools {
        jdk 'jdk17'
        maven 'maven'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'master', changelog: false, poll: false, url: 'https://github.com/Omarrh49/juice-shop.git'
            }
        }
        
        stage('Code Compile') {
            steps {
                sh "mvn clean compile"
            }
        }
        
        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --format XML', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        
        stage('Sonar Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=juice-shop \
                    -Dsonar.java.binaries=. \
                    -Dsonar.projectKey=juice-shop'''
                }
            }
        }
        
        stage('Trivy Scan') {
            steps {
                sh "trivy fs . > trivy-fs_report.txt"
            }
        }
        
        stage('Code Build') {
            steps {
                sh "mvn clean install"
            }
        }
        
        stage('Docker Login') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                        sh "echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin"
                    }
                }
            }
        }
        
        stage('Docker Build') {
            steps {
                script {
                    sh "docker build -t juice-shop ."
                }
            }
        }
        
        stage('Trivy') {
            steps {
                sh "trivy image omarrh/juice-shop:tagname > trivy.txt"
            }
        }
        
        stage('Docker Push') {
            steps {
                script {
                    sh "docker tag juice-shop omarrh/juice-shop:tagname"
                    sh "docker push omarrh/juice-shop:tagname"
                }
            }
        }
        
         stage('Deploy') {
            steps {
                  sh "docker run -d --name juice-shop -p 5000:5000 omarrh/juice-shop:tagname"

            }
        }
      
           stage ("Docker run Dastardly from Burp Suite Scan") {
            steps {
                cleanWs()
                sh '''
                    docker run --user $(id -u) -v ${WORKSPACE}:${WORKSPACE}:rw \
                    -e BURP_START_URL=https://juice-shop.herokuapp.com/ \
                    -e BURP_REPORT_FILE_PATH=${WORKSPACE}/dastardly-report.xml \
                    public.ecr.omarrh/juice-shop:tagname
                '''
            }
        }
    }
}
