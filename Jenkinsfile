pipeline {
    agent any

  environment {
    registry = "mm167/numeric-app"
    registryCredential = 'docker-hub'
    deploymentName = "devsecops"
    containerName = "devsecops-container"
    serviceName = "devsecops-svc"
    imageName = "mm167/numeric-app:${BUILD_NUMBER}"
    applicationURL = "http://192.168.49.2"
    applicationURI = "/increment/99"
  }

    stages {
        stage('GitHub') {
            steps {
                // Get some code from a GitHub repository
                git(
                    url: "https://github.com/user20201901/kubernetes-devops-security.git",
                    branch: "main",
                    changelog: true,
                    poll: true
                )
            }
        }
        
        stage('Build Artifact') {
            steps {
              sh "mvn clean package -DskipTests=true"
              archive 'target/*.jar' //so that they can be downloaded later
            }
        } 
        
        stage('Unit Tests - JUnit and Jacoco') {
          steps {
            sh "mvn test"
          }
        }
      	stage('Mutation Tests - PIT') {
	      steps {
	        sh "mvn org.pitest:pitest-maven:mutationCoverage"
	      }
          post {
            always {
              pitmutation mutationStatsFile: '**/target/pit-reports/**/mutations.xml'
            }
          }	
	    } 
        stage('SonarQube - SAST') {
	      steps {
	        withSonarQubeEnv('SonarQube') {
	           sh "mvn clean verify sonar:sonar -Dsonar.projectKey=numeric-application -Dsonar.projectName='numeric-application' -Dsonar.host.url=http://localhost:9000 "
	         }
	        timeout(time: 2, unit: 'MINUTES') {
	          script {
	            waitForQualityGate abortPipeline: true
	          }
	        }
	      }
        } 
        
	    stage('Vulnerability Scan - Docker') {
	      steps {
	        parallel(
	          "Dependency Scan": {
	            sh "mvn dependency-check:check"
	          },
	          "Trivy Scan": {
	            sh "bash trivy-docker-image-scan.sh"
	          },
	          "OPA Conftest": {
	            sh 'docker run --rm -v $(pwd):/project openpolicyagent/conftest test --policy opa-docker-security.rego dockerfiles/Dockerfile'
	          }
	        )
	      }
	    }
	       
        stage('Docker Build and Push') {
          steps {
            withDockerRegistry([credentialsId: "docker-hub", url: ""]) {
              sh 'printenv'
              sh 'cp target/*.jar dockerfiles'
              sh 'cd dockerfiles && docker build -t $registry:$BUILD_NUMBER .'
              sh 'docker push $registry:$BUILD_NUMBER'
            }
          }
        }
        
        stage('Remove Unused docker image') {
          steps{
            sh "docker rmi $registry:$BUILD_NUMBER"
          }
        }
        stage('Vulnerability Scan - Kubernetes') {
          steps {
	    parallel(
	      "OPA Scan": {
	        sh 'docker run --rm -v $(pwd):/project openpolicyagent/conftest test --policy opa-k8s-security.rego k8s_deployment_service.yaml'
	      },
	      "Kubesec Scan": {
	        sh "bash kubesec-scan.sh"
	      },
	      "Trivy Scan": {
	        sh "bash trivy-k8s-scan.sh"
	      }
	    )
          }
        }      
        stage('Kubernetes Deployment - DEV') {
          steps {
            withKubeConfig([credentialsId: 'kubernetes-config']) {
              sh "bash k8s-deployment.sh"
            }
          }
        }
        stage('Kubernetes Rollout') {
          steps {
            withKubeConfig([credentialsId: 'kubernetes-config']) {
              sh "bash k8s-deployment-rollout-status.sh"
            }
          }
        }
    }
    
  post {
    always {
      junit 'target/surefire-reports/*.xml'
      jacoco execPattern: 'target/jacoco.exec'
      dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
    }

    // success {

    // }

    // failure {

    // }
  }
}
