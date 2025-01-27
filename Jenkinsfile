@Library('slack') _
pipeline {
  agent any

  environment {
    deploymentName = "devsecops"
    containerName = "devsecops-container"
    serviceName = "devsecops-svc"
    imageName = "harish4948/numeric-app:${GIT_COMMIT}"
    applicationURL="http://34.172.227.128"
    applicationURI="/increment/99"
  }
  stages {
      stage('Build Artifact') {
            steps {
              sh "mvn clean package -DskipTests=true" 
              archive 'target/*.jar'
            }
        }   
      stage('Unit Tests') {
            steps {
              sh "mvn test"
            }
        } 
        stage('Mutation Tests') {
            steps {
              sh "mvn org.pitest:pitest-maven:mutationCoverage"
            }
        }
        stage('SonarQube') {
            environment {
              token = credentials('sonarqube-token')
              // ip = credentials('sonarqube-ip')
            }
            steps {
              withSonarQubeEnv('Sonarqube'){
              sh 'mvn sonar:sonar -Dsonar.projectKey=numeric-application -Dsonar.host.url=http://localhost:9000'
              }
              timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
           } 
           stage('Vulnerability Scan - Docker') {
            steps {
              parallel(
                "Dependency Scan":{
             sh "mvn dependency-check:check"},
            "Trivy Scan":{
              sh "bash trivy-docker-image-scan.sh"
            },
            "OPA conftest":{
              sh 'docker run --rm -v $(pwd):/project openpolicyagent/conftest test --policy opa-docker-security.rego Dockerfile'
            }
              )
          }
      }
          stage('Docker Build and push') {
            steps {
              withDockerRegistry([credentialsId:"docker-hub", url: ""]){
              sh "printenv"
              sh 'sudo docker build -t harish4948/numeric-app:""$GIT_COMMIT"" .'
              sh 'docker push harish4948/numeric-app:""$GIT_COMMIT""'
              }}
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
           
    stage('K8S Deployment - DEV') {
      steps {
        parallel(
          "Deployment": {
            withKubeConfig([credentialsId: 'kubeconfig']) {
              sh "bash k8s-deployment.sh"
            }
          },
          "Rollout Status": {
            withKubeConfig([credentialsId: 'kubeconfig']) {
              sh "bash k8s-deployment-rollout-status.sh"
            }
          }
        )
      }
    }
        stage('Integration Tests - PROD') {
      steps {
        script {
          try {
            withKubeConfig([credentialsId: 'kubeconfig']) {
              sh "bash integration-test-PROD.sh"
            }
          } catch (e) {
            withKubeConfig([credentialsId: 'kubeconfig']) {
              sh "kubectl -n prod rollout undo deploy ${deploymentName}"
            }
            throw e
          }
        }
      }
    }
       stage('OWASP ZAP - DAST') {
      steps {
        withKubeConfig([credentialsId: 'kubeconfig']) {
          sh 'bash zap.sh'
          
        }
      }
    }
       stage('Prompte to PROD?') {
      steps {
        timeout(time: 2, unit: 'DAYS') {
          input 'Do you want to Approve the Deployment to Production Environment/Namespace?'
        }
      }
    }
    stage('K8S CIS Benchmark') {
      steps {
        script {

          parallel(
            "Master": {
              sh "bash cis-master.sh"
            },
            "Etcd": {
              sh "bash cis-etcd.sh"
            },
            "Kubelet": {
              sh "bash cis-kubelet.sh"
            }
          )

        }
      }
    }
       stage('K8S Deployment - PROD') {
      steps {
        parallel(
          "Deployment": {
            withKubeConfig([credentialsId: 'kubeconfig']) {
              sh 'sed -i "s#replace#${imageName}#g" k8s_PROD_deployment_service.yaml'
              sh 'kubectl -n prod apply -f k8s_PROD_deployment_service.yaml'
            }
          },
          "Rollout Status": {
            withKubeConfig([credentialsId: 'kubeconfig']) {
              sh "bash k8s-deployment-rollout-status.sh"
            }
          }
        )
      }
    }
}
post{
  always{
junit 'target/surefire-reports/*.xml'
jacoco execPattern: 'target/jacoco.exec'
pitmutation mutationStatsFile: '**/target/pit-reports/**/mutations.xml'
dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
publishHTML([allowMissing: false, alwaysLinkToLastBuild: true, keepAll: true, reportDir: 'owasp-zap-report', reportFiles: 'zap_report.html', reportName: 'OWASP ZAP HTML REPORT', reportTitles: 'OWASP ZAP HTML REPORT', useWrapperFileDirectly: true])
sendNotifications currentBuild.result
  }
}
}
