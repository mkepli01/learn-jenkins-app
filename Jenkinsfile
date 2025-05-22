pipeline {
    agent any

    environment{
        NETLIFY_SITE_ID = '2b2df2ee-7bbf-4866-8df7-10f3eb06539b'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
        REACT_APP_VERSION = "1.0.$BUILD_ID"
    }

    stages {

        stage('Docker'){
            steps{
                sh 'docker build -t my-playwright .'
            }
        }
        stage('Build') {
            agent {
                docker{
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    echo "small change"
                    ls -la
                    node --version
                    npm --version
                    npm ci 
                    npm run build
                    ls -la
                '''
            }
        }
        stage('Run Tests'){
            parallel{
                stage('Test'){
                    agent {
                        docker{
                            image 'node:18-alpine'
                            reuseNode true
                        }
                    }
                    steps{
                        sh'''
                            echo "Running tests..."
                            test -f build/index.html || exit 1
                            npm test
                        '''
                    }

                    post{
                        always{
                            junit 'jest-results/junit.xml'
                        }
                    }
                }
                
                stage('E2E'){
                    agent {
                        docker{
                            image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                            reuseNode true
                        }
                    }
                    steps{
                        sh'''
                            npm install serve
                            node_modules/.bin/serve -s build &
                            sleep 10
                            npx playwright test --reporter=html
                        '''
                    }

                    post{
                        always{
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright Local Report', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
                }
            }
        }

        stage('Deploy Staging'){
            agent {
                docker{
                    image 'my-playwright'
                    reuseNode true
                }
            }

            environment{
                CI_ENVIRONMENT_URL =  "STAGING_URL_TO_BE_REPLACED"
            }

            steps{
                sh'''
                    netlify --version
                    echo "Deploying to Staging... Site ID: $NETLIFY_SITE_ID"
                    netlify status
                    netlify deploy --dir build --json > deploy-output.json
                    CI_ENVIRONMENT_URL=$(node-jq -r '.deploy_url' deploy-output.json) 
                    npx playwright test --reporter=html
                '''
            }

            post{
                always{
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright E2E Staging Report', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }

        // stage('Approval'){
        //     steps{
        //         timeout(time: 15, unit: 'MINUTES'){
        //             input message: 'Deploy to Production?', ok: 'Yes, Deploy!'
        //         }
        //     }
        // }

        stage('Deploy Prod'){
            agent {
                docker{
                    image 'my-playwright'
                    reuseNode true
                }
            }


            environment{
                CI_ENVIRONMENT_URL =  'https://astounding-gingersnap-7b6b1f.netlify.app/'
            }
            steps{
                sh'''
                    netlify --version
                    echo "Deploying to Netlify... Site ID: $NETLIFY_SITE_ID"
                    netlify status
                    netlify deploy --prod --dir build
                    npx playwright test --reporter=html
                '''
            }

            post{
                always{
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright Prod E2E Report', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }
    }
}