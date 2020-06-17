def builderImage
def productionImage
def ACCOUNT_REGISTRY_PREFIX
def GIT_COMMIT_HASH

pipeline {
    agent any
    stages {
        stage('Checkout Source Code and Logging Into Registry') {
            steps {
                echo 'Logging Into the Private ECR Registry'
                script {
                    GIT_COMMIT_HASH = sh (script: "git log -n 1 --pretty=format:'%H'", returnStdout: true)
                    ACCOUNT_REGISTRY_PREFIX = "myregistry.domain.com"
                    sh """
                    #\$(aws ecr get-login --no-include-email --region us-east-1)
		      docker login -u jithu -p jithu myregistry.domain.com
                    """
                }
            }
        }

        stage('Make A Builder Image') {
            steps {
                echo 'Starting to build the project builder docker image'
                script {
                    builderImage = docker.build("${ACCOUNT_REGISTRY_PREFIX}/example-webapp-builder:${GIT_COMMIT_HASH}", "-f ./Dockerfile.builder .")
                    builderImage.push()
                    builderImage.push("${env.GIT_BRANCH}")
                    builderImage.inside('-v $WORKSPACE:/output -u root') {
                        sh """
                           cd /output
                           lein uberjar
                        """
                    }
                }
            }
        }

        stage('Unit Tests') {
            steps {
                echo 'running unit tests in the builder image.'
                script {
                    builderImage.inside('-v $WORKSPACE:/output -u root') {
                    sh """
                       cd /output
                       lein test
                    """
                    }
                }
            }
        }

        stage('Build Production Image') {
            steps {
                echo 'Starting to build docker image'
                script {
                    productionImage = docker.build("${ACCOUNT_REGISTRY_PREFIX}/example-webapp:${GIT_COMMIT_HASH}")
                    productionImage.push()
                    productionImage.push("${env.GIT_BRANCH}")
                }
            }
        }

 
        stage('Deploy to Production fixed server') {
            when {
                branch 'release'
            }
            steps {
                echo 'Deploying release to production'
                script {
                    productionImage.push("deploy")
                    sh """
                       #aws ec2 reboot-instances --region us-east-1 --instance-ids i-0e438e2bf64427c9d
			#cat << EOF > start-website
			echo 'docker login -u jithu -p jithu myregistry.domain.com' > start-website
			echo 'sudo docker rm my-website --force' >> start-website
			echo 'sudo docker run -d --rm -p 3000:3000 --name my-website ${ACCOUNT_REGISTRY_PREFIX}/example-webapp:release' >> start-website
			#EOF
			echo 'Created the executable file'
			echo '=========================='
			cat start-website
			echo '=========================='
			sudo mv start-website /var/lib/cloud/scripts/per-boot/start-website
			sudo chmod +x /var/lib/cloud/scripts/per-boot/start-website
			/var/lib/cloud/scripts/per-boot/start-website
                    """
                }
            }
        }


        stage('Integration Tests') {
            when {
                branch 'master'
            }
            steps {
                echo 'Deploy to test environment and run integration tests'
                script {
                    //TEST_ALB_LISTENER_ARN="arn:aws:elasticloadbalancing:us-east-1:089778365617:listener/app/testing-website/3a4d20158ad2c734/49cb56d533c1772b"
                    //sh """
                    //./run-stack.sh example-webapp-test ${TEST_ALB_LISTENER_ARN}
                    //"""

                    sh """
                        echo 'docker login -u jithu -p jithu myregistry.domain.com' > start-website-int
                        echo 'sudo docker rm my-website-int --force' >> start-website-int
                        echo 'sudo docker run -d --rm -p 4000:3000 --name my-website-int ${ACCOUNT_REGISTRY_PREFIX}/example-webapp:master' >> start-website-int
                        
			sudo mv start-website-int /var/lib/cloud/scripts/per-boot/start-website-int
                        sudo chmod +x /var/lib/cloud/scripts/per-boot/start-website-int
                        /var/lib/cloud/scripts/per-boot/start-website-int
                    """

                }
                echo 'Running tests on the integration test environment'
                script {
                    sh """
                       curl -v http://localhost:5000 | grep '<title>Welcome to example-webapp</title>'
                       if [ \$? -eq 0 ]
                       then
                           echo tests pass
                       else
                           echo tests failed
                           exit 1
                       fi
                    """
                }
            }
        }

 
        stage('Deploy to Production') {
            when {
                branch 'master'
            }
            steps {
                script {
                    productionImage.push("prod")
                    sh """
                        echo 'docker login -u jithu -p jithu myregistry.domain.com' > start-website-prod
                        echo 'sudo docker rm my-website-prod --force' >> start-website-prod
                        echo 'sudo docker run -d --rm -p 5000:3000 --name my-website-prod ${ACCOUNT_REGISTRY_PREFIX}/example-webapp:prod' >> start-website-prod
                        sudo mv start-website /var/lib/cloud/scripts/per-boot/start-website-prod
                        sudo chmod +x /var/lib/cloud/scripts/per-boot/start-website-prod
                        /var/lib/cloud/scripts/per-boot/start-website-prod
                    """
 
                   //PRODUCTION_ALB_LISTENER_ARN="arn:aws:elasticloadbalancing:us-east-1:089778365617:listener/app/production-website/a0459c11ab5707ca/5d21528a13519da6"
                    //sh """
                    //./run-stack.sh example-webapp-production ${PRODUCTION_ALB_LISTENER_ARN}
                    //"""
                }
            }
        }
    }
}
