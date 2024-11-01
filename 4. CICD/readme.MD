# CI/CD

**Requirements**
- Jenkins
- Sonarqube


# Steps

Pada tahapan selanjutnya untuk melakukan CI/CD, saya menggunakan Jenkins dalam membuat pipeline CI/CD.


# Staging Pipeline CI/CD

```
pipeline {
  agent any

  stages {
    stage (" Pull Repository Retail Store Sample ") {
      steps {
        script {
          sshagent(credentials: [SECRET]) {
            sh """ssh -o StrictHostKeyChecking=no ${MASTER_SERVER} << EOF
              cd ${MASTER_DIR_GIT}
              git checkout staging
              git pull testing staging
              exit
            EOF"""
          }
        }
      }
    }

    stage (" Code Analysis ") {
      environment {
        scannerHome = tool name: 'sonarqube'
      }

      steps {
        script {
          withSonarQubeEnv('sonarqube') {
            sh """
              ${scannerHome}/bin/sonar-scanner \
              -Dsonar.projectKey=Retail-Store-Helmfile \
              -Dsonar.sources=. \
              -Dsonar.exclusions=**/*.java \
              -Dsonar.host.url=${SONAR_URL} \
              -Dsonar.login=${SONAR_TOKEN}
            """
          }
        }
      }
    }

    stage (" Pull and Push Docker Images ") {
      steps {
        sshagent(credentials: [SECRET]) {
          sh """ssh -o StrictHostKeyChecking=no ${MASTER_SERVER} << EOF
            cd ${MASTER_DIR}
            docker pull public.ecr.aws/aws-containers/retail-store-sample-ui:0.8.2
            docker tag public.ecr.aws/aws-containers/retail-store-sample-ui:0.8.2 crocoxolen/retail-store-sample-ui:latest
            docker push crocoxolen/retail-store-sample-ui:latest
            docker system prune -a -f
            exit
          EOF"""
        }
      }
    }

    stage (" Deploy Apps using Helmfile ") {
      steps {
        sshagent(credentials: [SECRET]) {
          sh """ssh -o StrictHostKeyChecking=no ${MASTER_SERVER} << EOF
            cd ${HELMFILE_DIR}
            helmfile apply
            echo "Deploy Apps Success"
            exit
          EOF"""
        }
      }
    }
  }
}
```
![alt text](https://github.com/lawrence-olen/Test-Axiata/blob/master/4.%20CICD/images/axiata13.png?raw=true)

![alt text](https://github.com/lawrence-olen/Test-Axiata/blob/master/4.%20CICD/images/axiata16.png?raw=true)



# Production Pipeline CI/CD

```
pipeline {
  agent any

  stages {
    stage (" Pull Repository Retail Store Sample ") {
      steps {
        script {
          sshagent(credentials: [SECRET]) {
            sh """ssh -o StrictHostKeyChecking=no ${MASTER_SERVER} << EOF
              cd ${MASTER_DIR_GIT}
              git checkout production
              git pull testing production
              exit
            EOF"""
          }
        }
      }
    }

    stage ( "Code Analysis" ) {
      environment {
        scannerHome = tool name: 'sonarqube'
      }

      steps {
        script {
          withSonarQubeEnv('sonarqube') {
            sh """
              ${scannerHome}/bin/sonar-scanner \
              -Dsonar.projectKey=Retail-Store-Compose \
              -Dsonar.sources=. \
              -Dsonar.exclusions=**/*.java \
              -Dsonar.host.url=${SONAR_URL} \
              -Dsonar.login=${SONAR_TOKEN}
            """
          }
        }
      }
    }

    stage (" Pull and Push Docker Images ") {
      steps {
        sshagent(credentials: [SECRET]) {
          sh """ssh -o StrictHostKeyChecking=no ${MASTER_SERVER} << EOF
            cd ${MASTER_DIR}
            docker pull public.ecr.aws/aws-containers/retail-store-sample-catalog:0.8.2
            docker tag public.ecr.aws/aws-containers/retail-store-sample-catalog:0.8.2 crocoxolen/retail-store-sample-catalog:latest
            docker push crocoxolen/retail-store-sample-catalog:latest
            docker system prune -a -f
            exit
          EOF"""
        }
      }
    }

    stage (" Deploy Apps using Docker Compose ") {
      steps {
        sshagent(credentials: [SECRET_PROD]) {
          sh """ssh -o StrictHostKeyChecking=no ${DOCKER_SERVER} << EOF
            cd ${COMPOSE_DIR}
            MYSQL_PASSWORD='testing' docker compose --file docker-compose.yml up -d
            echo "Deploy Apps Success"
            exit
          EOF"""
        }
      }
    }
  }
}
```

![alt text](https://github.com/lawrence-olen/Test-Axiata/blob/master/4.%20CICD/images/axiata14.png?raw=true)

![alt text](https://github.com/lawrence-olen/Test-Axiata/blob/master/4.%20CICD/images/axiata17.png?raw=true)



# Sonarqube

Disini saya menggunakan Sonarqube untuk melakukan testing terhadap repository staging maupun production.
![alt text](https://github.com/lawrence-olen/Test-Axiata/blob/master/4.%20CICD/images/axiata15.png?raw=true)


> [!NOTE]
> Beberapa Tahapan Pipeline CI/CD.
> - Mengambil repositori dari Github.
> - Menguji aplikasi menggunakan Sonarqube.
> - Pull dan Push docker image ke registry Docker Hub.
> - Mendeploy ulang aplikasi Retail Store.