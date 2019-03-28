pipeline {
    environment {
      DOCKER = credentials('docker-hub')
    }
  agent any
  stages {
// Building your Test Images
    stage('BUILD') {
      parallel {
        stage('Express Image') {
          steps {
            sh 'docker build -f express-image/Dockerfile \
            -t nodeapp-prod:nodeapp-prod .'
          }
        }
        stage('Test-Unit Image') {
          steps {
            sh 'docker build -f test-image/Dockerfile \
            -t test-image:nodeapp-prod .'
          }
        }
      }
      post {
        failure {
            echo 'This build has failed. See logs for details.'
        }
      }
    }
// Performing Software Tests
    stage('TEST') {
      parallel {
        stage('Mocha Tests') {
          steps {
            sh 'docker run --name nodeapp-prod --network="bridge" -d \
            -p 9000:9000 nodeapp-prod:nodeapp-prod'
            sh 'docker run --name test-image -v $PWD:/JUnit --network="bridge" \
            --link=nodeapp-prod -d -p 9001:9000 \
            test-image:nodeapp-prod'
          }
        }
        stage('Quality Tests') {
          steps {
            sh 'docker login --username $DOCKER_USR --password $DOCKER_PSW'
            sh 'docker tag nodeapp-prod:nodeapp-prod humayunalam/nodeapp-prod:nodeapp-prod'
            sh 'docker push humayunalam/nodeapp-prod:nodeapp-prod'
          }
        }
      }
      post {
        success {
            echo 'Build succeeded.'
        }
        unstable {
            echo 'This build returned an unstable status.'
        }
        failure {
            echo 'This build has failed. See logs for details.'
        }
      }
    }
// Deploying your Software
    stage('DEPLOY') {
          when {
           branch 'master'  //only run these steps on the master branch
          }
            steps {
                    retry(3) {
                        timeout(time:10, unit: 'MINUTES') {
                            sh 'docker tag nodeapp-prod:nodeapp-prod humayunalam/nodeapp-prod:nodeapp-prod'
                            sh 'docker push humayunalamnodeapp-prod:nodeapp-prod'
                            sh 'docker save humayunalam/nodeapp-prod:nodeapp-prod | gzip > nodeapp-prod-golden.tar.gz'
                        }
                    }

            }
            post {
                failure {
                    sh 'docker stop nodeapp-prod test-image'
                    sh 'docker system prune -f'
                    deleteDir()
                }
            }
    }
// JUnit reports and artifacts saving
    stage('REPORTS') {
      steps {
        junit 'reports.xml'
        archiveArtifacts(artifacts: 'reports.xml', allowEmptyArchive: true)
        archiveArtifacts(artifacts: 'nodeapp-prod-golden.tar.gz', allowEmptyArchive: true)
      }
    }
// Doing containers clean-up to avoid conflicts in future builds
    stage('CLEAN-UP') {
      steps {
        sh 'docker stop nodeapp-prod test-image'
        sh 'docker system prune -f'
        deleteDir()
      }
    }
  }
}
