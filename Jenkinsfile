@Library('Shared') _

pipeline {
    agent any
    
    environment {
        // Update the main app image name to match the deployment file
        DOCKER_IMAGE_NAME = 'almsys/easyshop-app'
        DOCKER_MIGRATION_IMAGE_NAME = 'almsys/easyshop-migration'
        DOCKER_IMAGE_TAG = "${BUILD_NUMBER}"
        GITHUB_CREDENTIALS = credentials('github-credentials')
        GIT_BRANCH = "master"
        HOST = 'easyshop.local.lab'
    }
    
    stages {
        stage('Cleanup Workspace') {
            steps {
                script {
                    clean_ws()
                }
            }
        }
        
        stage('Clone Repository') {
            steps {
                script {
                    clone("https://github.com/sysops8/tws-e-commerce-app_hackathon.git","master")
                }
            }
        }
        
        stage('Build Docker Images') {
            parallel {
                stage('Build Main App Image') {
                    steps {
                        script {
                            docker_build(
                                imageName: env.DOCKER_IMAGE_NAME,
                                imageTag: env.DOCKER_IMAGE_TAG,
                                dockerfile: 'Dockerfile',
                                context: '.'
                            )
                        }
                    }
                }
                
                stage('Build Migration Image') {
                    steps {
                        script {
                            docker_build(
                                imageName: env.DOCKER_MIGRATION_IMAGE_NAME,
                                imageTag: env.DOCKER_IMAGE_TAG,
                                dockerfile: 'scripts/Dockerfile.migration',
                                context: '.'
                            )
                        }
                    }
                }
            }
        }
        
        stage('Run Unit Tests') {
            steps {
                script {
                    run_tests()
                }
            }
        }
        
        stage('Security Scan with Trivy') {
            steps {
                script {
                    // Create directory for results
                  
                    trivy_scan()
                    
                }
            }
        }
        
        stage('Push Docker Images') {
            parallel {
                stage('Push Main App Image') {
                    steps {
                        script {
                            docker_push(
                                imageName: env.DOCKER_IMAGE_NAME,
                                imageTag: env.DOCKER_IMAGE_TAG,
                                credentials: 'docker-hub-credentials'
                            )
                        }
                    }
                }
                
                stage('Push Migration Image') {
                    steps {
                        script {
                            docker_push(
                                imageName: env.DOCKER_MIGRATION_IMAGE_NAME,
                                imageTag: env.DOCKER_IMAGE_TAG,
                                credentials: 'docker-hub-credentials'
                            )
                        }
                    }
                }
            }
        }
        
        // Add this new stage
        stage('Update Kubernetes Manifests') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'github-credentials', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
                        sh """
                            echo "=== Updating image versions ==="
                            
                            # Обновляем основной образ
                            sed -i 's|image: almsys/easyshop-app:.*|image: almsys/easyshop-app:${env.DOCKER_IMAGE_TAG}|g' kubernetes/08-easyshop-deployment.yaml
                            
                            # Обновляем migration образ
                            sed -i 's|image: almsys/easyshop-migration:.*|image: almsys/easyshop-migration:${env.DOCKER_IMAGE_TAG}|g' kubernetes/07-migration.yaml
                            
                            git config user.email "almastvx@gmail.com"
                            git config user.name "Jenkins CI"
                            git add kubernetes/
                            git commit -m "Deploy easyshop version '${env.DOCKER_IMAGE_TAG}'"
                            git push https://$GIT_USER:$GIT_TOKEN@github.com/sysops8/tws-e-commerce-app_hackathon.git HEAD:master
                        """
                    }
                }
            }
        }
}
