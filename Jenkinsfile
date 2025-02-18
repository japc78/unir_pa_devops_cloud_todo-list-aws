pipeline {
    agent any

    stages {
        stage('Get Code') {
            steps {
                showEnvironmentInfo()
                git branch: 'master', credentialsId: 'unir.devops.github.token', url: 'https://github.com/japc78/unir_pa_devops_cloud_todo-list-aws.git'
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

        stage('Deploy') {
            steps {
                echo 'Starting deploy'
                unstash name: 'allProject'
                showEnvironmentInfo()
                sh '''
                    sam build
                    sam deploy --config-file samconfig.toml \
                        --config-env production \
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
                BASE_URL = getStackOutput("todo-list-aws-production", "BaseUrlApi")
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
                        pytest -m read_only --junitxml=junit-rest.xml test/integration/todoApiTest.py
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