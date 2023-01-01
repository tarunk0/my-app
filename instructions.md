## Dockerizing the React Application:

- Firstly, I created the react js application.
- Then wrote the Dockerfile to build the docker image.

```
FROM node:14-alpine
ENV NODE_ENV development
WORKDIR /app
COPY package.json .
RUN yarn install
COPY . .
EXPOSE 3000
CMD [ "yarn", "start"]

```

- Then built the image using docker build -t react-app:v1
- Then tried to run the application on to the docker host.

![](2023-01-01-05-24-44.png)

- We can see from the above screenshot that application is working fine on our local machine.
  

  ## Making the kubernetes cluster on GKE:

  - I used GCP to create the kubernetes cluster.
  
  ![](2023-01-01-14-35-29.png)

  - Installed Jenkins on this kubernetes Cluster using Helm Charts and also configured the Kuberntes pod as dynamic jenkins agent. So that our CICD pipeline can run on the jenkins dynamic pods and after the pipeline is completed the pod will be terminated.
  
```
        pipeline {
        agent {
            kubernetes {
            yaml '''
                apiVersion: v1
                kind: Pod
                metadata:
                labels:
                    app: test
                spec:
                containers:
                - name: git
                    image: bitnami/git:latest
                    command:
                    - cat
                    tty: true
                - name: kubectl-helm-cli
                    image: kunchalavikram/kubectl_helm_cli:latest
                    command:
                    - cat
                    tty: true
                - name: kaniko
                    image: gcr.io/kaniko-project/executor:debug
                    command:
                    - cat
                    tty: true
                    volumeMounts:
                    - name: kaniko-secret
                    mountPath: /kaniko/.docker
                volumes:
                - name: kaniko-secret
                    secret:
                    secretName: regcred
                    items:
                        - key: .dockerconfigjson
                        path: config.json
            '''
            }      
        }
        environment{
            DOCKERHUB_USERNAME = "tarunk0"
            APP_NAME = "react-webapp"
            IMAGE_NAME = "${DOCKERHUB_USERNAME}" + "/" + "${APP_NAME}"
            IMAGE_TAG = "${BUILD_NUMBER}"
            PROJECT_ID = 'round-forge-355507'
            CLUSTER_NAME = 'tk-cluster'
            LOCATION = 'us-central1-a'
            CREDENTIALS_ID = 'gke'	
        }
        stages {
            stage('Checkout SCM') {
            steps {
                container('git') {
                git url: 'https://github.com/tarunk0/my-app.git',
                branch: 'main'
                }
            }
            }
        
        stage('Build and push Container Image'){
            steps {
            container('kaniko'){
                sh "/kaniko/executor --context $WORKSPACE --destination $IMAGE_NAME:$IMAGE_TAG --destination $IMAGE_NAME:latest"
            }
            }
        }   
        stage('Deploy to K8s') {
                    steps{
                    container('kubectl-helm-cli') {
                        echo "Deployment started ..."
                        sh 'ls -ltr'
                        sh 'pwd'
                        sh "sed -i 's/tagversion/${env.BUILD_ID}/g' service.yaml"
                        sh "sed -i 's/tagversion/${env.BUILD_ID}/g' deployment.yaml"
                        echo "Starting deployment of service.yaml"
                        step([$class: 'KubernetesEngineBuilder', projectId: env.PROJECT_ID, clusterName: env.CLUSTER_NAME, location: env.LOCATION, manifestPattern: 'service.yaml', credentialsId: env.CREDENTIALS_ID, verifyDeployments: true])
                        echo "Starting deployment of deployment.yaml"
                        step([$class: 'KubernetesEngineBuilder', projectId: env.PROJECT_ID, clusterName: env.CLUSTER_NAME, location: env.LOCATION, manifestPattern: 'deployment.yaml', credentialsId: env.CREDENTIALS_ID, verifyDeployments: true])
                        echo "Deployment Finished ..."
                    }
                    }
                }
            }
        }

```

- We can all the steps followed inside the Jenkins file in the above code.
- In the first part of the code we can see the kuberntes pods which has been configured to run every stage inside the respective containers. For Eg: We have Kaniko container for basically building and pushing the docker image to docker hub.
- In last step we can see that, The application is being deployed to the kubernetes cluster which is GKE and for doing that I have installed a plugin called GKE inside jenkins and using that Plugin We can deploy the application on to the cluster.
- We need Service Account's json key to basically authenticate ourselves with the cluster. I added the Credentials inside jenkins of type Private key from GCP by the name of GKE


![](2023-01-01-14-47-22.png)

- I have attached the screenshots after the pipeline has run successfully.
- Also we have the LoadBalancer provisioned for acessing the application.
- Please the screenshots.

![image](https://user-images.githubusercontent.com/92631457/210166433-791d7dc6-2688-40e1-b486-56a0c416eeaa.png)

![image](https://user-images.githubusercontent.com/92631457/210166453-46cd8778-a852-4164-aaa8-cf4db3ac4323.png)

- In the above screenshot you can see that 4 containers are running in a pod for running the various stages inside k8s only.

![image](https://user-images.githubusercontent.com/92631457/210166491-4b213e1d-ce01-46f7-bd4f-81c82031ba71.png)

- We can see after the pipeline has run successfully the image has been pushed to Docker hub.

![image](https://user-images.githubusercontent.com/92631457/210166527-642f08b8-ea6d-48a9-8d4e-c73445910ea7.png)

- You can see from the above screenshot that, I also have set up the webhook trigger inside github to send the data back to jenkins for automatically triggering the pipeline after the commit has happened or lets say the PR has been opened.

![image](https://user-images.githubusercontent.com/92631457/210166606-0ba099b8-41f3-4e8e-a125-9956fc09b091.png)

- Above screenshot shows that the pipeline has run successfully and the required no of replicas has also been deployed onto the cluster.

![image](https://user-images.githubusercontent.com/92631457/210166643-1d0ae727-e102-4309-a98e-bb13f9c881f1.png)

- We can see that the pods are running and also the load balancer has been provisioned.
- The Containers in which our pipeline was running are also terminating after the pipeline has completed.

![image](https://user-images.githubusercontent.com/92631457/210166725-2729f093-12d1-49a1-b487-6c2d11798769.png)

- We can access the application using the Loadbalancer's IP.
- Next time as soon as the commit has been pushed to github the pipeline will run automatically as we have setup github webhook trigger and new docker image will be formed and then pushed to docker hub and the new changes will be applied to the application, using the same loadbalancer IP we can access the application.

- I believe I have finished both the tasks of Firstly dockerizing the application and then making the CICD pipeline using the dockerized application.


