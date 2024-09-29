pipeline {

    agent any

	tools {
        jdk 'jdk17'
        maven "maven3"
    }
 
    environment {
        registry = "marcuskoh95/gitops-proj"
        registryCredential = 'dockerhub-cred'
        AWS_ACCESS_KEY_ID = credentials('AWS_ACCESS_KEY_ID')
        AWS_SECRET_ACCESS_KEY = credentials('AWS_SECRET_ACCESS_KEY')

        gitRepo = 'https://github.com/marcuskohtongjie95/vprofile-project-docker.git'
        branchName = 'cicd-kube'
        githubCredentials = credentials('GITHUB_TOKEN') // Use Jenkins Credentials to store the GitHub PAT

    }

    triggers {
        // Triggers the build when changes are pushed to GitHub
        githubPush()
    }

    stages{

        stage('Git Checkout'){
            steps {
                git branch: "${branchName}", url: "${gitRepo}"
            }
        }


        stage('BUILD'){
            steps {
                sh 'mvn clean install -DskipTests'
            }
            post {
                success {
                    echo 'Now Archiving...'
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }
        }

        stage('UNIT TEST'){
            steps {
                sh 'mvn test'
            }
        }

        /*stage('Trivy scan fs') {
            steps{
              script {
                sh "trivy fs --scanners vuln --format table -o trivy-fs-report.html ."
              }
            }
        }*/

        stage('INTEGRATION TEST'){
            steps {
                sh 'mvn verify -DskipUnitTests'
            }
        }

        stage ('CODE ANALYSIS WITH CHECKSTYLE'){
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Generated Analysis Result'
                }
            }
        }


        stage('Building image') {
            steps{
              script {
                dockerImage = docker.build registry + ":$BUILD_NUMBER"
              }
            }
        }

        
        /*stage('Trivy scan image') {
            steps{
              script {
                sh "trivy image ${registry}:$BUILD_NUMBER -o trivy-image-report.html"
                /*sh "trivy image --exit-code 1 --severity HIGH,CRITICAL ${registry}:$BUILD_NUMBER"
                

              }
            }
        }*/
        
        stage('Upload Image') {
          steps{
            script {
              docker.withRegistry( '', registryCredential ) {
                dockerImage.push("$BUILD_NUMBER")
                dockerImage.push('latest')
              }
            }
          }
        }

        stage('Update Helm values.yaml') {
            steps {
                script {
                    // Update the Helm chart values.yaml file with the latest Docker image tag
                    sh """
                        sed -i 's|appimage:\\s*tag:.*|appimage:\\n  tag: ${BUILD_NUMBER}|' helm/vprofilecharts/values.yaml
                    """
                }
            }
        }
        
        stage('Commit and Push Changes after Helm values.yaml is updated') {
            steps {
                script {
                    // Configure git with Jenkins CI user credentials and commit the changes
                    sh """
                    git config user.email "marcuskoh95@gmail.com"
                    git config user.name "marcuskoh95"
                    git add helm/vprofilecharts/values.yaml
                    git commit -m "Updated appimage tag to ${BUILD_NUMBER}"
                    git push https://"${githubCredentials}"@github.com/marcuskohtongjie95/vprofile-project-docker.git "${branchName}"
                    """
                    
                }
            }
        }

        stage('Remove Unused docker image') {
          steps{
            sh "docker rmi $registry:$BUILD_NUMBER"
          }
        }

        stage('CODE ANALYSIS with SONARQUBE') {

            environment {
                scannerHome = tool 'sonar-scanner'
            }

            steps {
                withSonarQubeEnv('sonarqube-server') {
                    sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                   -Dsonar.projectName=vprofile-repo \
                   -Dsonar.projectVersion=1.0 \
                   -Dsonar.sources=src/ \
                   -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                   -Dsonar.junit.reportsPath=target/surefire-reports/ \
                   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }

                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        /*
        stage('Kubernetes Deploy') {
	  agent { label 'KOPS' }
            steps {
                    sh "helm upgrade --install --force vproifle-stack helm/vprofilecharts --set appimage=${registry}:${BUILD_NUMBER} --namespace prod"
            }
        }
        */

        stage('Deploy to EKS') {
            steps {
                script{
                    sh "aws eks update-kubeconfig --name gitops-proj-eks --region us-east-1"

                    // Helm deploy command
                    sh '''
                    helm upgrade --install my-app-release helm/vprofilecharts \
                        --namespace default \
                        --set appimage.repository=$registry \
                        --set appimage.tag=$BUILD_NUMBER
                    '''
                    }
                }
            }

        stage('Clean Up Docker Images') {
            steps {
                sh "docker rmi $registry:$BUILD_NUMBER || true"
                sh "docker rmi $registry:latest || true"
                }
            }
        }

    }
