pipeline {
    agent any

  environment {
    registry = "mm167/numeric-app"
    registryCredential = 'docker-hub'
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
	    } 
        stage('SonarQube - SAST') {
	      steps {
	        withSonarQubeEnv('SonarQube') {
	           sh "mvn clean verify sonar:sonar -Dsonar.projectKey=numeric-application -Dsonar.projectName='numeric-application' -Dsonar.host.url=http://localhost:9000 -Dsonar.token=sqp_2cf746e1a5f986afef77df28c9adc48fee626529"
	         }
	        timeout(time: 2, unit: 'MINUTES') {
	          script {
	            waitForQualityGate abortPipeline: true
	          }
	        }
	      }
        } 
	    stage('Vulnerability Scan - Docker ') {
	      steps {
	        sh "mvn dependency-check:check"
	      }
	    }    
        stage('Docker Build and Push') {
          steps {
            withDockerRegistry([credentialsId: "docker-hub", url: ""]) {
              sh 'printenv'
              sh 'docker build -t $registry:$BUILD_NUMBER .'
              sh 'docker push $registry:$BUILD_NUMBER'
              
            }
          }
        }
        
        stage('Remove Unused docker image') {
          steps{
            sh "docker rmi $registry:$BUILD_NUMBER"
          }
        }
        
        stage('Kubernetes Deployment - DEV') {
          steps {
            withKubeConfig([credentialsId: 'kubernetes-config']) {
              sh "sed -i 's#replace#${registry}:${BUILD_NUMBER}#g' k8s_deployment_service.yaml"
              sh "kubectl apply -f k8s_deployment_service.yaml"
            }
          }
        }
    }
    
  post {
    always {
      junit 'target/surefire-reports/*.xml'
      jacoco execPattern: 'target/jacoco.exec'
      pitmutation mutationStatsFile: '**/target/pit-reports/**/mutations.xml'
      dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
    }

    // success {

    // }

    // failure {

    // }
  }
}
