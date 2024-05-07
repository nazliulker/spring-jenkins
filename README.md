# Springboot web uygulamasına şu şekillerde ulaşabilirsiniz;
## Curl ile istek atabiliriz.
```
curl --header 'Host: springjenkins.com' http://135.181.199.177
```
## Hosts file'a ekleyebiliriz. Ardından http://springjenkins.com adresini ziyaret edebilirsiniz.

```
135.181.199.177 springjenkins.com
```


# Springboot ile ekrana "Hello World :)" yazdıran bir web uygulaması yazdım.
# Kubernetes Cluster oluşturdum.
# Jenkins kurdum.
# Webhook ile Jenkins ve Github bağlantısını kurdum. Böylece her committe CI/CD süreci otomatik olarak başlıyor.
# Deployment.yaml ve service.yaml oluşturdum.
### deployment.yaml 
### Her deploy işleminde podun yeniden başlaması için strategy type: "Recreate" olarak ayarladım.
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: nazlinesibeulker/devops-integration:latest
        imagePullPolicy: Always
        resources:
          limits:
            memory: "512Mi"
            cpu: "500m"
        ports:
        - containerPort: 8090
  
```
### service.yaml
### Cluster'a güvenli bir şekilde ulaşabilmek için myapp service type ClusterIP olmalıdır.
```
---
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  type: ClusterIP
  selector:
    app: myapp
  ports:
  - name: http
    port: 80
    targetPort: 8090
```

# Jenkins file oluşturdum.

```
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
```
## Git pluginin checkout komutu ile kodu çekip mvn ile derledim.
```
        stage('Build Image'){
            steps{
                script{
                     sh 'docker buildx build --platform linux/amd64 -t nazlinesibeulker/devops-integration:v1.$BUILD_ID .'
                     sh 'docker buildx build --platform linux/amd64 -t nazlinesibeulker/devops-integration:latest .'
                 }
            }
        }
```
## jdk17 kullandığım için buildx --platform komutu ile image'ı build ettim.
```
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
```
## Build aldığım image'ı docker credantial ile docker hub'a pushladım.
```
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
```
## Build id'yi deploymentservice.yaml'a değişken olarak atadım. kubernetesDeploy komutu ile cluster'a deploy ettim.

# Nginx-ingress-controller ve ingress oluşturdum.

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/cloud/deploy.yaml

```
## ingress.yaml
```
 apiVersion: networking.k8s.io/v1
 kind: Ingress
 metadata:
   name: myingress
   labels:
     name: myingress
   annotations:
    kubernetes.io/ingress.class: nginx
 spec:
   rules:
   - host: springjenkins.com
     http:
       paths:
       - pathType: Prefix
         path: "/"
         backend:
           service:
             name: myapp
             port:
               name: http
```
## Ingress'in dışardan gelen trafiği myapp backend servisine yönlendirecek şekilde ayarladım. Böylece http://springjenkins.com hostuna dışardan erişelebilir hale geldi.

