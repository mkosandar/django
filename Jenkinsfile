pipeline {
    agent any
    tools {
        //gradle "7.4.2"
        //jdk "JDK15"
        maven "mvn 3.9.6"
        dockerTool "docker"
        nodejs "nodejs"
    }
    environment {     
    DOCKERHUB_CREDENTIALS= credentials('dockerHub-login')
    PRIVATE_KEY= credentials('mk_server')
    }
    stages {
        stage("Build") {
            steps {
                script{
                    sh 'pip install -r requirements.txt '
                    sh 'npm install'
                }
            }
        }
        /*stage("Unit-test") {
            steps {
                script{
                     echo "mvn test" 
                }
            }
        }*/
        stage("detect-secret") {
            steps {
                script{
                    //sh "/var/lib/jenkins/.local/bin/detect-secrets scan"
                    //sh "/var/lib/jenkins/.local/bin/detect-secrets scan --all-files --force-use-all-plugins > detect-secrets-report.json', returnStatus: true"
                    def secretScan = sh(script: '/var/lib/jenkins/.local/bin/detect-secrets scan > detect-secrets-report.json', returnStatus: true)
                    if (secretScan != 0) {
                        error("Secrets detected in the codebase!")
                    } else {
                        echo "No secrets detected."
                    }
                }
            }
        }
        /*stage("snyk") {
            steps {
                script{
                    snykSecurity additionalArguments: '--file=requirements.txt', failOnIssues: false, snykInstallation: 'snyk', snykTokenId: 'snyk-token'
                }
            }
        }
        stage('SonarQube analysis') {
            steps {
                script {
                    def scannerHome = tool 'sonarqube-scanner';
                    withSonarQubeEnv('sonarqube') {
                        sh "${scannerHome}/bin/sonar-scanner -e -Dsonar.projectName=webgoat -Dsonar.projectKey=webgoat -Dsonar.language=java -Dsonar.sources=src/main/java -Dsonar.java.binaries=target/classes -Dsonar.scm.disabled=true"
                    }
                    waitForQualityGate abortPipeline: false
                 }
             }
         }
        stage("Checkmarx") {
            steps {
                script{
                    checkmarxASTScanner additionalOptions: '', baseAuthUrl: '', branchName: '', checkmarxInstallation: 'Checkmarx', credentialsId: '', projectName: '${env.JOB_NAME}', serverUrl: '', tenantName: ''
                }
            }
        }
        stage('Checkov') {
            steps {
                script {
                    docker.image('bridgecrew/checkov:latest').inside("--entrypoint=''") {
                        sh 'checkov -d . --output-file-path checkov_report.json --output json '
                     }
                 }
             }
         }*/
        stage('Secret Detection') {
            steps {
                script {
                    def truffleStatus = sh(script: "trufflehog --regex --entropy=True ./", returnStatus: true)
                    def detectSecretsStatus = sh(script: "detect-secrets scan", returnStatus: true)
                    if (truffleStatus != 0 || detectSecretsStatus != 0) {
                        error "Secrets detected"
                    }
                }
            }
        }
        /*stage("docker-publish") {
            steps {
                script{
                    sh """
                    docker build -t mayureshkosandar/webgoat:1.0 .
                    docker login -u ${DOCKERHUB_CREDENTIALS_USR} -p ${DOCKERHUB_CREDENTIALS_PSW}
                    docker push mayureshkosandar/webgoat:1.0
                    """
                }
            }
        }*/
        stage('OWASP Dependency-Check Vulnerabilities') {
            steps {
                dependencyCheck additionalArguments: ''' 
                    -o './'
                    -s './'
                    -f 'ALL' 
                    --prettyPrint''', odcInstallation: 'DependencyCheck'
                dependencyCheckPublisher failedNewCritical: 5, failedTotalCritical: 5, pattern: 'dependency-check-report.xml'
                }
          }
        /*stage("deploy-to-devsecops") {
            steps {
                sshPublisher(publishers: [sshPublisherDesc(configName: 'devsecops', transfers: [sshTransfer(cleanRemote: false, excludes: '', execCommand: "docker rm -f webgoat; docker run -dit -p 9988:8080 --name webgoat mayureshkosandar/webgoat:1.0 ", execTimeout: 120000, flatten: false, makeEmptyDirs: false, noDefaultExcludes: false, patternSeparator: '[, ]+', remoteDirectory: '', remoteDirectorySDF: false, removePrefix: '', sourceFiles: '')], usePromotionTimestamp: false, useWorkspaceInPromotion: false, verbose: true)])
            }
        }
        stage('prod-deployment'){
            agent {
                docker {
                    image 'alpine:latest'
                    args ' -u root '
                }
            }
            environment {
                SSH_CREDENTIALS_ID = 'mk_server' 
                HOST_IP = '192.168.92.114' 
            }
            steps{
                sh """
                apk update
                apk add --no-cache openssh-client
                ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -P ""
                cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
                """
                sshagent([env.SSH_CREDENTIALS_ID]) {
                    sh"""
                    unset SSH_AUTH_SOCK;
                    unset SSH_AGENT_PID;
                    ssh -o StrictHostKeyChecking=no mk@${env.HOST_IP} "docker run -dit -p 9090:8080 --name webgoat mayureshkosandar/webgoat:1.0"
                    """
                }
            } 
        }
        stage('prod-deployment') {
            agent {
                docker {
                    image 'alpine:latest'
                    args ' -u root '
                    // Run the container on the node specified at the
                    // top-level of the Pipeline, in the same workspace,
                    // rather than on a new node entirely:
                    //reuseNode true
                }
            }
            steps {
                sh """
                apk update
                apk add --no-cache openssh-client
                sshagent(['mk_server']) {
                    sh 'ssh mk@192.168.92.114 "docker run -dit -p 9090:8080 --name webgoat mayureshkosandar/webgoat:1.0"'
                }
                echo $PRIVATE_KEY >> /tmp/id_rsa
                chmod 600 /tmp/id_rsa
                ssh -i /tmp/id_rsa -tt mk@192.168.92.114
                docker run -dit -p 9090:8080 --name webgoat mayureshkosandar/webgoat:1.0
                """
                }
        }
        stage("prod-deployment") {
            steps {
                script{
                    docker.image('alpine:latest').inside('-u root -v $SSH_AUTH_SOCK:/ssh-agent -e SSH_AUTH_SOCK=/ssh-agent') {
                        sh """
                        apk update
                        apk add --no-cache openssh-client
                        ssh -T mk@192.168.92.114
                        docker run -dit -p 9090:8080 --name webgoat mayureshkosandar/webgoat:1.0
                        """
                    }
                    //sh """
                    //docker rm -f webgoat
                    //docker run -dit -p 9090:8080 --name webgoat mayureshkosandar/webgoat:1.0 
                    //"""
                }
            }
        }*/
    }
}
