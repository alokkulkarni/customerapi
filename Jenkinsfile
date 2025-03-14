pipeline {
    agent any
    environment {
        DOCKER_REGISTRY = 'https://docker.io/alokkulkarni'  // Replace with your registry
        IMAGE_NAME = 'customerapi'
        IMAGE_TAG = "${BUILD_NUMBER}"
        // SONAR_PROJECT_KEY = 'customerapi'
        // AWS_REGION = 'eu-west-2'  // Adjust as needed
        // S3_BUCKET = 'customerapi'  // Replace with your S3 bucket
        // KUBECONFIG = credentials('eks-kubeconfig')  // Jenkins credential ID for kubeconfig
        // SCAN_S3_BUCKET = 'your-security-reports-bucket'  // Add this line
        GITHUB_REPO = 'alokkulkarni/customerapi'  // Replace with your GitHub org/repo
        GITHUB_BRANCH = 'master'  // Replace with your default branch
    }
    tools {
        jdk 'JDK 17'  // Make sure this matches your Jenkins tool configuration
    }
    stages {
        stage('Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: "*/${GITHUB_BRANCH}"]],
                    extensions: [],
                    userRemoteConfigs: [[
                        credentialsId: 'github-app',
                        url: "https://github.com/alokkulkarni/paymentsapi.git"
                    ]]
                ])
            }
        }
        stage('Build') {
            steps {
                sh 'chmod +x ./gradlew'
                sh './gradlew clean build -x test'
            }
        }
        // stage('Test') {
        //     steps {
        //         script {
        //             def workspace = pwd()
        //             sh """
        //                 cd ${workspace}
        //                 ./gradlew test jacocoTestReport
        //                 ./gradlew pitest
        //             """
        //             junit allowEmptyResults: true, testResults: '**/build/test-results/test/*.xml'
        //             jacoco(
        //                 execPattern: "${workspace}/build/jacoco/test.exec",
        //                 classPattern: "${workspace}/build/classes/java/main",
        //                 sourcePattern: "${workspace}/src/main/java",
        //                 exclusionPattern: "${workspace}/src/test/*"
        //             )
        //         }
        //     }
        // }
        stage('Generate SBOM') {
            steps {
                script {
                    def workspace = pwd()
                    sh """
                        cd ${workspace}
                        ./gradlew cyclonedxBom
                        mkdir -p sbom-artifacts
                        cp build/sbom/* sbom-artifacts/
                    """
                    archiveArtifacts artifacts: 'sbom-artifacts/*', fingerprint: true
                }
            }
        }
        stage('Analyze SBOM for Vulnerabilities') {
            steps {
                // Use Grype or other tool to scan SBOM for vulnerabilities
                grypeScan autoInstall: true, repName: 'grypeReport_${JOB_NAME}_${BUILD_NUMBER}.txt', scanDest: 'dir:sbom-artifacts'
                // sh '''
                //     if command -v grype &> /dev/null; then
                //       echo "Scanning dependencies for vulnerabilities..."
                //       mkdir -p vulnerability-reports
                //       grype dir:sbom-artifacts -o json > vulnerability-reports/grype-report.json
                //       grype dir:sbom-artifacts -o table > vulnerability-reports/grype-report.txt
                //     else
                //       echo "Grype not available, skipping vulnerability scan"
                //       mkdir -p vulnerability-reports
                //       echo "Vulnerability scan skipped - Grype not installed" > vulnerability-reports/scan-skipped.txt
                //     fi
                // '''
                // Archive vulnerability reports
                archiveArtifacts artifacts: 'vulnerability-reports/*', fingerprint: true
            }
        }
        // stage('SonarQube Analysis') {
        //     steps {
        //         withSonarQubeEnv('SonarQube') {  // Configure this in Jenkins
        //             sh """
        //                 ./gradlew sonar \
        //                 -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
        //                 -Dsonar.java.binaries=build/classes/java/main \
        //                 -Dsonar.coverage.jacoco.xmlReportPaths=build/reports/jacoco/test/jacocoTestReport.xml \
        //                 -Dsonar.pitest.reportsDirectory=build/reports/pitest
        //             """
        //         }
        //     }
        // }
        // stage('Export Reports to S3') {
        //     steps {
        //         withAWS(region: "${AWS_REGION}", credentials: 'aws-credentials') {  // Configure AWS credentials in Jenkins
        //             sh """
        //                 aws s3 cp build/test-results/test s3://${S3_BUCKET}/${BUILD_NUMBER}/test-reports/ --recursive
        //                 aws s3 cp build/reports/jacoco s3://${S3_BUCKET}/${BUILD_NUMBER}/coverage-reports/ --recursive
        //                 aws s3 cp build/reports/pitest s3://${S3_BUCKET}/${BUILD_NUMBER}/mutation-reports/ --recursive
        //                 aws s3 cp sbom-artifacts s3://${S3_BUCKET}/${BUILD_NUMBER}/sbom/ --recursive
        //                 aws s3 cp vulnerability-reports s3://${S3_BUCKET}/${BUILD_NUMBER}/vulnerability-reports/ --recursive
        //             """
        //         }
        //     }
        // }
        stage('Build and Push Docker Image') {
            steps {
                script {
                    docker.withRegistry("${DOCKER_REGISTRY}", 'docker-credentials') {  // Configure Docker credentials in Jenkins
                        def customImage = docker.build("${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}")
                        // Add SBOM attestation to the image if cosign is available
                        sh '''
                            if command -v cosign &> /dev/null; then
                              echo "Attaching SBOM attestation to image..."
                              cosign attach sbom --sbom sbom-artifacts/bom.json ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
                            else
                              echo "Cosign not available, skipping SBOM attestation"
                            fi
                        '''
                        customImage.push()
                        customImage.push('latest')
                    }
                }
            }
        }
        stage('Container Security Scan') {
            steps {
                script {
                    def workspace = pwd()
                    def scanScript = "${workspace}/scripts/container-scan.sh"
                    sh """
                        cd ${workspace}
                        ${scanScript} -i "${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}" \
                                    -b "${SCAN_S3_BUCKET}" \
                                    -f "json" \
                                    -s "HIGH,CRITICAL" \
                                    -R "${AWS_REGION}"
                    """
                }
            }
        }
        // stage('Deploy to EKS') {
        //     steps {
        //         script {
        //             def workspace = pwd()
        //             withKubeConfig([credentialsId: 'eks-kubeconfig']) {
        //                 sh """
        //                     cd ${workspace}
        //                     helm upgrade --install customerapi helm/customerapi \
        //                         --namespace dev \
        //                         --create-namespace \
        //                         --set image.repository=${DOCKER_REGISTRY}/${IMAGE_NAME} \
        //                         --set image.tag=${IMAGE_TAG} \
        //                         --set sbom.enabled=true \
        //                         -f helm/customerapi/values-dev.yaml
        //                 """
        //             }
        //         }
        //     }
        // }
    }
    post {
        always {
            script {
                if (getContext(hudson.FilePath)) {
                    deleteDir()
                }
            }
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}