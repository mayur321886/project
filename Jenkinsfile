//pipeline code
pipeline {
    agent any
    stages {
        stage('Install Dependencies') {
            steps {
                sh 'cd $WORKSPACE'
                sh 'rm -rf project || true'
                git branch: "master",
                    url: "https://github.com/mayur321886/project"
                sh 'ls'
            }
        }
        stage('Pre-Build Tests') {
            parallel {
                stage('Git Repository Scanner'){
                    steps {
                        sh 'cd $WORKSPACE'
                        sh 'trufflehog https://github.com/mayur321886/project --json | jq "{branch:.branch, commitHash:.commitHash, path:.path, stringsFound:.stringsFound}" > trufflehog.report || true'
                        archiveArtifacts artifacts: 'trufflehog_report.json'
                        sh 'cat trufflehog_report.json'
                        sh 'echo "Scanning Repositories.....done"'
                    }
                }
                stage('Image Security') {
                    steps {
                        sh 'cd $WORKSPACE'
                        sh 'dockle --input ~/docker_img_backup/prod_tomcat.tar -f json -o mytomcat_report.json'
                        sh 'cat prod_tomcat_report.json | jq {summary}'
                        sh 'dockle --input ~/docker_img_backup/pgadmin4.tar -f json -o pgadmin4_report.json'
                        sh 'cat pgadmin4_report.json | jq {summary}'
                        sh 'dockle --input ~/docker_img_backup/postgres11.6.tar -f json -o postgres11_report.json'
                        sh 'cat postgres11_report.json | jq {summary}'
                        sh 'dockle --input ~/docker_img_backup/zap2docker-stable.tar -f json -o zap2docker-stable_report.json'
                        sh 'cat zap2docker-stable_report.json | jq {summary}'
                        sh 'dockle --input ~/docker_img_backup/sonarqube.tar -f json -o sonarqube_report.json'
                        sh 'cat sonarqube_report.json | jq {summary}'
                        archiveArtifacts artifacts: '*.json'
                    }
                }
            }
        }
        stage('Build Stage') {
            steps {
                sh 'mvn clean'
                sh 'mvn compile'
                sh 'mvn install package'
            }
        }
        stage('Initializing Docker') {
            steps {
                sh 'docker-compose up -d'
                sh 'docker build -t prod_tomcat .'
                sh 'docker run --name login --rm  --network project_project -p 80:8080 -d prod_tomcat'
                sh 'docker run --name sonar_for_tomcat --rm --network project_project -p 4444:9000 -d owasp/sonarqube' 
            }
        }
        stage('SonarQube Analysis') {
            steps {
                sh 'mvn sonar:sonar -Dsonar.projectKey=mayur -Dsonar.host.url=http://localhost:4444 -Dsonar.login=2f3e1c66f6fc180a6df64adf6a5729c8e686e4af || true'
            }
        }
        stage('SCA') {
            parallel {
                stage('Dependency Check') {
                    steps {
                        sh 'wget https://github.com/mayur321886/project/blob/master/dc.sh'
                        sh 'chmod +x dc.sh'
                        archive (includes: 'dependency-check-report.html')
                        archive (includes: 'dependency-check-report.html')
                    }
                }
                stage('Junit Testing') {
                    steps {
                        archive (includes: 'dependency-check-junit.xml')
                    }
                }
            }
        }
        stage('DAST') {
            steps {
                sh 'docker run --name dast_full --network project_project -t owasp/zap2docker-stable zap-full-scan.py -t http://localhost/LoginWebApp/ -J report_dast_full.json'
                sh 'docker run --name dast_baseline --network project_project -v /root/:/zap/wrk/:rw -t owasp/zap2docker-stable zap-baseline.py -t http://localhost/LoginWebApp/ -J report.json --autooff'
            }
        }
    }
}
