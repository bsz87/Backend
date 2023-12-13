def imageName="bsz87/backend"
def dockerTag=""

def dockerRegistry=""
def registryCredentials="dockerhub"

pipeline
{
    agent
    {
        label 'agent'
    }
    environment 
    {
        PIP_BREAK_SYSTEM_PACKAGES = "1"
        scannerHome = tool 'SonarQube'
    }
    stages
    {
        stage('Hello')
        {
            steps
            {
                echo 'Hello World'
            }
        }
        stage('Get code from Github')
        {
            steps
            {
                checkout scm
            }
        }
        stage('Unit tests')
        {
            steps
            {
                sh "pip3 install -r requirements.txt"
                sh "python3 -m pytest --cov=. --cov-report xml:test-results/coverage.xml --junitxml=test-results/pytest-report.xml"
            }
        }
        stage('Sonarqube analysis')
        {
            steps
            {
                withSonarQubeEnv('SonarQube') 
                {
                    sh "${scannerHome}/bin/sonar-scanner"
                }
            }
        }
        stage('Docker build image') 
        {
            steps 
            {
                script 
                {
                  dockerTag = "RC-${env.BUILD_ID}.${env.GIT_COMMIT.take(7)}"
                  applicationImage = docker.build("$imageName:$dockerTag",".")
                }
            }
        }
        stage ('Push Docker image to Artifactory repo') 
        {
            steps 
            {
                script 
                {
                    docker.withRegistry("$dockerRegistry", "$registryCredentials") 
                    {
                        applicationImage.push()
                        applicationImage.push('latest')
                    }
                }
            }
        }
        stage ('Push to Repo') {
            steps {
                dir('ArgoCD') {
                    withCredentials([gitUsernamePassword(credentialsId: 'git', gitToolName: 'Default')]) 
                    {
                        git branch: 'master', url: 'https://github.com/bsz87/ArgoCD.git'
                        sh """ cd backend
                        git config --global user.email "b.szelagowski@gmail.com"
                        git config --global user.name "bsz87"
                        sed -i "s#$imageName.*#$imageName:$dockerTag#g" backend_deploy.yaml
                        git commit -am "Set new $dockerTag tag."
                        git diff
                        git push origin master
                        """
                    }
                } 
            }
        } 
    }
    post 
    {
        always 
        {
            junit testResults: "test-results/*.xml"
            cleanWs()
        }
        // success 
        // {
        //     build job: 'app_of_apps', parameters: [ string(name: 'backendDockerTag', value: "$dockerTag")], wait: false
        // }
    }
}