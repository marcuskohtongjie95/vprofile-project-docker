## Intro
Purpose: GitOps project to set up Multi-Tier Web application Stack and deploy onto EKS with NGINX as ingress controller/laod balancer

This repo contains 
1. Jenkinsfile for GitOps 
2. Helm charts to deploy onto EKS
3. Source code for web app stack written in Java
    - Tomcat Server running web application server
    - MySQL DB storing login details
    - RabbitMQ connected to Tomcat server (dummy service)

## Helm Charts
1. Deployment files
    - vproapp (customized docker image for tomcat web app server)
    - vprodb (mysql docker image)
    - rmq (rmq docker image)
2. Service files
    - vproapp-service (ClusterIP)
    - vprodb-service (ClusterIP)
    - rmq-service (ClusterIP)
3. Secret file
    - store DB and RMQ password (base64 encoded)
4. Ingress file
    - NGINX ingress (routing to vproapp service on port 80)
    - Assumption that NGINX ingress controller has already been manually deployed (one-time installation required) onto EKS cluster

## Jenkins Pipeline
- Git checkout
- Sonarscanner statis code analysis (bugs, security vulnerabilities, duplication, code coverage), then upload results to SonarQube server where quality gate will return pass result if threshold for code quality is ok.
- Maven Build & Unit Test for java source code, which returns .war artifact
- Build customized docker image (base image: Tomcat), using .war artifact from earlier stage to upload into Tomcat server.
- Utilise Trivy to scan docker image for vulnerabilities
- Customized docker image is uploaded to Dockerhub
- Helm values.yaml is updated with latest docker image tag, then committed and pushed to github
- Remove unused docker image on Jenkins server to save space
- Helm chart is deployed to EKS (can use shell command on jenkins server i.e. helm upgrade OR replace this part with ArgoCD)


## Prerequisites
- JDK 1.8 or later
- Maven 3 or later
- MySQL 5.6 or later
- Jenkins server already setup on EC2 instance (minimum t2.medium)
- SonarQube server aleady setup on EC2 instance (minimum t2.medium)
- EKS server already setup (using Terraform, refer to iac-vprofile repo)
- ArgoCD already deployed on EKS cluster using Helm Chart (e.g. upon detecting Helm chart has been updated in Git repo, EKS cluster is sync-ed with the new configuration and changes are automatically deployed)
- Prometheus and Grafana are already deployed on EKS Cluster using Helm Chart


## Technologies 
- Spring MVC
- Spring Security
- Spring Data JPA
- Maven
- JSP
- MySQL

## Database
Here,we used Mysql DB 
MSQL DB Installation Steps for Linux ubuntu 14.04:
- $ sudo apt-get update
- $ sudo apt-get install mysql-server

Then look for the file :
- /src/main/resources/accountsdb
- accountsdb.sql file is a mysql dump file.we have to import this dump to mysql db server
- > mysql -u <user_name> -p accounts < accountsdb.sql


