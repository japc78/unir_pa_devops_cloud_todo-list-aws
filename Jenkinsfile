pipeline {
    agent any

    stages {
        stage('Get Code') {
            steps {
                showEnvironmentInfo()
                git branch: 'develop', credentialsId: 'unir.devops.github.token', url: 'https://github.com/japc78/unir_pa_devops_cloud_todo-list-aws.git'
                echo 'Save stash'
                stash includes: '**', name: 'allProject'
                stash includes: 'src/*.py', name: 'pyFiles'
                stash includes: 'test/integration/*.py, pytest.ini', name: 'testIntegration'
            }
            post() {
                always {
                    echo "Cleaning up workspace..."
                    deleteDir()
                }
            }
        }

        stage('Static Test') {
            steps {
                unstash 'pyFiles'
                echo 'Static Test'
                showEnvironmentInfo()
                catchError(buildResult: 'FAILURE', stageResult: 'SUCCESS') {
                    sh '''
                        flake8 --format=pylint --exit-zero src > flake8.out
                        bandit --exit-zero -r ./src -f custom -o bandit.out --msg-template '{abspath}:{line}: [{test_id}] {msg}'
                    '''
                }
            }

            post() {
                always {
                    recordIssues(
                        tools: [
                            flake8(name: 'Flake8', pattern: 'flake8.out'),
                            pyLint(name: 'Bandit', pattern: 'bandit.out')
                        ]
                    )
                    echo "Cleaning up workspace..."
                    deleteDir()
                }
            }
        }

        stage('Deploy') {
            steps {
                echo 'Starting deploy'
                unstash name: 'allProject'
                showEnvironmentInfo()
                sh '''
                    sam build
                    sam deploy --config-file samconfig.toml \
                        --config-env staging \
                        --no-confirm-changeset \
                        --no-disable-rollback \
                        --no-fail-on-empty-changeset \
                        --resolve-s3
                '''
            }

            post() {
                always {
                    echo "Cleaning up workspace..."
                    deleteDir()
                }
            }
        }

        stage('Rest Test') {
            environment {
                BASE_URL = getStackOutput("todo-list-aws-staging", "BaseUrlApi")
            }

            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    unstash name: 'testIntegration'
                    showEnvironmentInfo()
                    waitForServiceAvailability("${env.BASE_URL}/todos", "API TodoList")

                    sh '''
                        echo "Base url es: $BASE_URL"
                        export PYTHONPATH=$WORKSPACE
                        echo "PYTHONPATH is set to: $PYTHONPATH"
                        pytest --junitxml=junit-rest.xml test/integration/todoApiTest.py
                    '''

                    junit('junit-rest.xml')
                }
            }
            post() {
                always {
                    echo "Cleaning up workspace..."
                    deleteDir()
                }
            }
        }

        stage('Promote') {
            environment {
                GIT_CREDENTIALS = credentials('unir.devops.github.token')
            }
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    echo 'Starting Promote'
                    showEnvironmentInfo()

                    git branch: 'develop', credentialsId: 'unir.devops.github.token', url: 'https://github.com/japc78/unir_pa_devops_cloud_todo-list-aws.git'

                    sh '''
                        git checkout master
                        git merge develop
                    '''

                    sh('git push https://$GIT_CREDENTIALS_USR:$GIT_CREDENTIALS_PSW@github.com/japc78/unir_pa_devops_cloud_todo-list-aws.git master')
                }
            }

            post {
                always {
                    echo "Cleaning up workspace..."
                    deleteDir()
                }
            }
        }
    }

    post() {
        always {
            echo "Cleaning up workspace..."
            deleteDir()
        }
    }
}

def waitForServiceAvailability(String url, String serviceName, int retries = 5, int delaySeconds = 5) {
    for (int attempt = 1; attempt <= retries; attempt++) {
        try {
            def status = sh(script: "curl -s -o /dev/null -w '%{http_code}' ${url}", returnStdout: true).trim()
            int responseCode = status.isInteger() ? status.toInteger() : 0

            if (responseCode in 200..299) {
                echo "${serviceName} is running (HTTP ${responseCode})."
                return
            } else {
                echo "Attempt ${attempt}: ${serviceName} responded with code: ${responseCode}, retrying in ${delaySeconds} seconds..."
            }
        } catch (Exception e) {
            echo "Attempt ${attempt}: Failed to connect to ${serviceName}, retrying in ${delaySeconds} seconds..."
        }

        if (attempt < retries) {
            sleep delaySeconds
        }
    }
    error "ERROR, PIPELINE FAIL | ${serviceName} did not respond after ${retries} attempts"
}

def getStackOutput(stackName, outputKey) {
    def outputValue = sh(
        script: """aws cloudformation describe-stacks \
                    --stack-name ${stackName} \
                    --query "Stacks[0].Outputs[?OutputKey=='${outputKey}'].OutputValue" \
                    --output text""",
        returnStdout: true
    ).trim()

    if (outputValue == "None" || outputValue == "") {
        error "No se encontró el OutputKey '${outputKey}' en el stack '${stackName}'"
    } else {
        return outputValue
    }
}

def showEnvironmentInfo() {
    def envInfo = sh(script: '''
        echo "Whoami: $(whoami)"
        echo "Hostname: $(hostname)"
        echo "WORKSPACE: ${WORKSPACE}"
    ''', returnStdout: true).trim()

    echo envInfo
    getNodeAndExecutorDetails()
}

def getNodeAndExecutorDetails() {
    def nodeName = env.NODE_NAME ?: 'main'
    def executorNumber = env.EXECUTOR_NUMBER
    def buildUrl = env.BUILD_URL
    def buildNumber = env.BUILD_NUMBER

    echo "### Node and Executor Details ###"
    echo "Node Name: ${nodeName}"
    echo "Executor Number: ${executorNumber}"
    echo "Build Number: ${buildNumber}"
    echo "Build URL: ${buildUrl}"
    echo "###############################"
}