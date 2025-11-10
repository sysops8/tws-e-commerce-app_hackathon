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
        
        stage('Update Kubernetes Manifests') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'github-credentials', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
                        sh """
                            echo "=== Updating Kubernetes manifests ==="
                            
                            # Проверяем текущие файлы
                            echo "Current directory:"
                            pwd
                            ls -la kubernetes/
                            
                            # Обновляем основной образ в deployment
                            echo "Updating easyshop-app image to version: ${env.DOCKER_IMAGE_TAG}"
                            if [ -f kubernetes/08-easyshop-deployment.yaml ]; then
                                sed -i 's|image: almsys/easyshop-app:.*|image: almsys/easyshop-app:${env.DOCKER_IMAGE_TAG}|g' kubernetes/08-easyshop-deployment.yaml
                                echo "✅ Updated 08-easyshop-deployment.yaml"
                            else
                                echo "❌ File kubernetes/08-easyshop-deployment.yaml not found!"
                                exit 1
                            fi
                            
                            # Обновляем migration образ в job
                            echo "Updating easyshop-migration image to version: ${env.DOCKER_IMAGE_TAG}"
                            if [ -f kubernetes/12-migration-job.yaml ]; then
                                sed -i 's|image: almsys/easyshop-migration:.*|image: almsys/easyshop-migration:${env.DOCKER_IMAGE_TAG}|g' kubernetes/12-migration-job.yaml
                                echo "✅ Updated 12-migration-job.yaml"
                            elif [ -f kubernetes/07-migration.yaml ]; then
                                sed -i 's|image: almsys/easyshop-migration:.*|image: almsys/easyshop-migration:${env.DOCKER_IMAGE_TAG}|g' kubernetes/07-migration.yaml
                                echo "✅ Updated 07-migration.yaml"
                            else
                                echo "⚠️  Migration job file not found, skipping..."
                            fi
                            
                            # Показываем изменения
                            echo "=== Changes made ==="
                            git diff kubernetes/ || true
                            
                            # Проверяем что изменения применились
                            echo "=== Verifying changes ==="
                            grep "image: almsys/easyshop-app:${env.DOCKER_IMAGE_TAG}" kubernetes/08-easyshop-deployment.yaml && echo "✅ Main app image updated correctly" || echo "❌ Main app image update failed"
                            
                            # Коммитим и пушим
                            echo "=== Committing and pushing changes ==="
                            git config user.email "almastvx@gmail.com"
                            git config user.name "Jenkins CI"
                            git add kubernetes/
                            git status
                            git commit -m "CI: Update image tags to version ${env.DOCKER_IMAGE_TAG} [build ${env.BUILD_NUMBER}]"
                            
                            echo "Pushing to GitHub..."
                            git push https://$GIT_USER:$GIT_TOKEN@github.com/sysops8/tws-e-commerce-app_hackathon.git HEAD:master
                            
                            echo "✅✅✅ Kubernetes manifests updated and pushed successfully!"
                            echo "🚀 ArgoCD should automatically detect changes and deploy new version"
                        """
                    }
                }
            }
        }       
    }
    
    post {
        always {
            script {
                echo "Build ${env.BUILD_NUMBER} completed"
                echo "Docker images:"
                echo "  - ${env.DOCKER_IMAGE_NAME}:${env.DOCKER_IMAGE_TAG}"
                echo "  - ${env.DOCKER_MIGRATION_IMAGE_NAME}:${env.DOCKER_IMAGE_TAG}"
                echo "Git commit: Updated to version ${env.DOCKER_IMAGE_TAG}"
            }
        }
        success {
            script {
                echo "✅ Pipeline completed successfully!"
                echo "📦 New version ${env.DOCKER_IMAGE_TAG} deployed"
            }
        }
        failure {
            script {
                echo "❌ Pipeline failed!"
                echo "Check logs for details"
            }
        }
    }
}
