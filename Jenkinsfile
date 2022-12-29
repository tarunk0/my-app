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

