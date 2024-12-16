pipeline {
    agent any
    environment {
        NEXUS_USER = credentials('nexus-username')
        NEXUS_PASSWORD = credentials('nexus-password')
        NEXUS_REPO = credentials('nexus-repo')
        BASTION_HOST = credentials('bastion-ip')
        ANSIBLE_HOST = credentials('ansible-ip')
    }
    stages {
        stage('Code Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh 'mvn sonar:sonar'
                }   
            }
        }
        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        stage('Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '--format HTML --format XML', nvdCredentialsId: 'nvd-key', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('Build Artifact') {
            steps {
                sh 'mvn clean package -DskipTests -Dcheckstyle.skip'
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $NEXUS_REPO/myapp .'
            }
        }
        stage('Push Artifact to Nexus Repo') {
            steps {
                nexusArtifactUploader artifacts: [[artifactId: 'spring-petclinic',
                classifier: '',
                file: 'target/spring-petclinic-2.4.2.war',
                type: 'war']],
                credentialsId: 'nexus-cred',
                groupId: 'Petclinic',
                nexusUrl: 'nexus.noektech.com',
                nexusVersion: 'nexus3',
                protocol: 'https',
                repository: 'nexus-repo',
                version: '1.0'
            }
        }
        stage('Log Into Nexus Docker Repo') {
            steps {
                sh 'docker login --username $NEXUS_USER --password $NEXUS_PASSWORD $NEXUS_REPO'
            }
        }
        stage('Push to Nexus Docker Repo') {
            steps {
                sh 'docker push $NEXUS_REPO/myapp'
            }
        }
        stage('Trivy image Scan') {
            steps {
                sh "trivy image $NEXUS_REPO/myapp > trivyfs.txt"
            }
        }
        stage('Delete image from Jenkins server') {
            steps {
                sh 'docker rmi $NEXUS_REPO/myapp'
            }
        }
        stage('Deploy to stage') {
            steps {
                sshagent(['ansible-key']) {
                    sh '''
                      ssh -t -t -o StrictHostKeyChecking=no -o ProxyCommand="ssh -W %h:%p -o StrictHostKeyChecking=no ec2-user@$BASTION_HOST" ec2-user@$ANSIBLE_HOST "ansible-playbook -i /etc/ansible/stage_hosts /etc/ansible/deployment.yml"
                    '''
                }
            }
        }
        stage('Check Stage Website Availability') {
            steps {
                 sh "sleep 90"
                 sh "curl -k -s -o /dev/null -w \"%{http_code}\" https://stage.noektech.com"
                script {
                    def response = sh(script: "curl -k -s -o /dev/null -w \"%{http_code}\" https://stage.noektech.com", returnStdout: true).trim()
                    if (response == "200") {
                        slackSend(color: 'good', message: "The stage petclinic java application is up and running with HTTP status code ${response}.", tokenCredentialId: 'slack')
                    } else {
                        slackSend(color: 'danger', message: "The stage petclinic java application appears to be down with HTTP status code ${response}.", tokenCredentialId: 'slack')
                    }
                }
            }
        }
        stage('Request for Approval') {
            steps {
                timeout(activity: true, time: 10) {
                    input message: 'Needs Approval ', submitter: 'admin'
                }
            }
        }
        stage('Deploy to Prod') {
            steps {
                sshagent(['ansible-key']) {
                    sh '''
                      ssh -t -t -o StrictHostKeyChecking=no -o ProxyCommand="ssh -W %h:%p -o StrictHostKeyChecking=no ec2-user@$BASTION_HOST" ec2-user@$ANSIBLE_HOST "ansible-playbook -i /etc/ansible/prod_hosts /etc/ansible/deployment.yml"
                    '''
                }
            }
        }
        stage('Check Prod Website Availability') {
            steps {
                 sh "sleep 90"
                 sh "curl -s -o /dev/null -w \"%{http_code}\" https://prod.noektech.com"
                script {
                    def response = sh(script: "curl -s -o /dev/null -w \"%{http_code}\" https://prod.noektech.com", returnStdout: true).trim()
                    if (response == "200") {
                        slackSend(color: 'good', message: "The prod petclinic java application is up and running with HTTP status code ${response}.", tokenCredentialId: 'slack')
                    } else {
                        slackSend(color: 'danger', message: "The prod petclinic java application appears to be down with HTTP status code ${response}.", tokenCredentialId: 'slack')
                    }
                }
            }
        }
    }
}
