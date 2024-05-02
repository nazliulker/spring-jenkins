pipeline {
    agent any

    stages{
        stage('Build Maven'){
            steps{
                checkout([$class: 'GitSCM', branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[url: 'https://github.com/nazliulker/spring-jenkins']]])
                withMaven {
                sh 'mvn clean install'
                }
            }
        }
        
        stage('Build Image'){
            steps{
                script{
                     sh 'docker buildx build --platform linux/amd64 -t nazlinesibeulker/devops-integration:v1.$BUILD_ID .'
                     sh 'docker buildx build --platform linux/amd64 -t nazlinesibeulker/devops-integration:latest .'
                 }
            }
        }
        stage('Push image to Hub'){
             steps{
                 script{
                   withCredentials([string(credentialsId: 'dockerhub-pwd', variable: 'dockerhubpwd')]) {
                    sh 'docker login -u nazlinesibeulker -p ${dockerhubpwd}.'
                   }
                    sh 'docker push nazlinesibeulker/devops-integration:v1.$BUILD_ID'
                    sh 'docker push nazlinesibeulker/devops-integration:latest'
                 }
             }
        }
        stage('Deploy to k8s'){
            steps{
                script{
		            sh '''
		            
		                tag=v1.$BUILD_ID
	  	                sed -i "s/latest/$tag/" deploymentservice.yaml
		             '''
		            print pwd
                    kubernetesDeploy (configs: 'deploymentservice.yaml',kubeconfigId: 'kubeconfigID')
                }
            }
        }
    }
}
