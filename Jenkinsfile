pipeline {
    agent any

    triggers {
        cron('0 9 * * *\n0 22 * * *')
    }

    environment {
        DOCKER_IMAGE = 'aziz/jenkins-tests:latest'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                echo '--- Building Docker image ---'
                script {
                    docker.build(env.DOCKER_IMAGE, '.')
                }
            }
        }

        stage('Run Tests in Docker') {
            steps {
                echo '--- Running tests in Docker ---'
                script {
                    docker.image(env.DOCKER_IMAGE).inside {
                        bat 'npm ci'

                        // Cypress
                        bat """
                            npx cypress run ^
                            --reporter mochawesome ^
                            --reporter-options ^
                            reportDir=reports\\\\cypress,reportFilename=index,overwrite=true,html=true,json=false,inlineAssets=true
                        """

                        // Newman
                        bat 'if not exist reports\\newman mkdir reports\\newman'
                        bat """
                            newman run MOCK_AZIZ_SERVEUR.postman_collection.json ^
                            -r html ^
                            --reporter-html-export reports\\\\newman\\\\newman-report.html
                        """

                        // K6
                        bat 'if not exist reports\\k6 mkdir reports\\k6'
                        bat 'k6 run test_k6.js --summary-export=reports\\k6\\summary.json'
                        bat 'python generate_k6_screenshot.py reports\\k6\\summary.json reports\\k6\\screenshot.png'
                    }
                }
            }
        }

        stage('Publish Report & Screenshot') {
            steps {
                publishHTML([
                    reportDir             : 'reports/cypress',
                    reportFiles           : 'index.html',
                    reportName            : 'Cypress Report',
                    keepAll               : true,
                    alwaysLinkToLastBuild : true,
                    allowMissing          : false
                ])

                publishHTML([
                    reportDir             : 'reports/newman',
                    reportFiles           : 'newman-report.html',
                    reportName            : 'Newman Report',
                    keepAll               : true,
                    alwaysLinkToLastBuild : true,
                    allowMissing          : false
                ])

                archiveArtifacts(
                    artifacts  : 'reports/k6/screenshot.png',
                    fingerprint: true
                )
            }
        }
    }

    post {
        always {
            echo 'Build enfin terminé.'
        }

        success {
            emailext subject: "Build Success: ${currentBuild.fullDisplayName}",
                      body: "Le build ${currentBuild.fullDisplayName} a réussi.\n\nConsultez les détails ici : ${env.BUILD_URL}",
                      to: 'aziztesteur@hotmail.com'
        }

        failure {
            emailext subject: "Build Failed: ${currentBuild.fullDisplayName}",
                      body: "Le build ${currentBuild.fullDisplayName} a échoué.\n\nConsultez les détail ici : ${env.BUILD_URL}",
                      to: 'aziztesteur@hotmail.com'
        }
    }
}

