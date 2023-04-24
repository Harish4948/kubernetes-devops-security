pipeline {
  agent any

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
              post { 
                always { 
                  junit 'target/surefire-reports/*.xml'
                  jacoco execPattern: 'target/jacoco.exec'
                }
              }
        } 
        stage('Mutation Tests') {
            steps {
              sh "mvn org.pitest:pitest-maven:mutationCoverage"
            }
              post { 
                always { 
                  pitmutation mutationStatsFile: '**/target/pit-reports/**/mutations.xml'
                }
              }
        }
          stage('Docker Build and push') {
            steps {
              withDockerRegistry([credentialsId:"docker-hub", url: ""]){
              sh "printenv"
              sh 'docker build -t harish4948/numeric-app:""$GIT_COMMIT"" .'
              sh 'docker push harish4948/numeric-app:""$GIT_COMMIT""'
              }}
           } 
           stage('SonarQube') {
            steps {
              withCredentials([credentialsId:"sonarqube-token"]){
              sh "echo ${SONARQUBE_TOKEN}"
              }}
           } 
           stage('Kubernetes Deploy - DEV') {
            steps {
              withKubeConfig([credentialsId:"kubeconfig"]){
              sh "sed -i 's#replace#harish4948/numeric-app:${GIT_COMMIT}#g' k8s_deployment_service.yaml"
              sh "kubectl apply -f k8s_deployment_service.yaml"
              }}
    }
}
}
