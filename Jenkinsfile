pipeline {

    agent any

    parameters {
        choice(name: 'TEST_TYPE', choices: ['LOAD', 'STRESS', 'SPIKE'], description: 'Select Test Type')
        choice(name: 'ENV', choices: ['DEV', 'QA', 'PROD'], description: 'Select Environment')

        string(name: 'THREAD_GROUPS', defaultValue: 'FishFlow', description: 'Comma-separated TGs')
        string(name: 'THREAD_CONFIG', defaultValue: 'FishFlow:5', description: 'Format → TG:Threads (FishFlow:10,DogFlow:20)')
    }

    environment {
        PT_REPO = 'https://github.com/ManishaS1713/MultiThreadGroupsInJmx_Jenkins.git'
    }

    stages {

        stage('Clean Workspace') {
            steps { cleanWs() }
        }

        stage('Checkout Code') {
            steps {
                git branch: 'main',
                    url: "${PT_REPO}",
                    credentialsId: 'PT_PipelineToken'
            }
        }

        stage('Validate Input') {
            steps {
                script {
                    if (!params.THREAD_GROUPS?.trim()) {
                        error "THREAD_GROUPS cannot be empty!"
                    }

                    env.SELECTED_TG = params.THREAD_GROUPS.replaceAll("\\s","")
                    env.THREAD_MAP = params.THREAD_CONFIG.replaceAll("\\s","")

                    echo "Selected TGs: ${env.SELECTED_TG}"
                    echo "Thread Config: ${env.THREAD_MAP}"
                }
            }
        }

        stage('Set Environment URL') {
            steps {
                script {
                    env.HOST = "petstore.octoperf.com"
                }
            }
        }

        stage('Run Performance Test') {
            steps {
                bat """
                IF EXIST result.jtl del result.jtl
                IF EXIST report rmdir /s /q report

                C:\\Jmeter\\apache-jmeter-5.6.3\\bin\\jmeter.bat -n ^
                -t MultiThreadGroupsInJmx_Jenkins.jmx ^
                -JthreadGroups=${env.SELECTED_TG} ^
                -JthreadConfig=${env.THREAD_MAP} ^
                -Jhost=${env.HOST} ^
                -l result.jtl ^
                -e -o report
                """
            }
        }

        stage('Publish Report') {
            steps {
                publishHTML([
                    reportDir: 'report',
                    reportFiles: 'index.html',
                    reportName: 'JMeter Report'
                ])
            }
        }
    }
}
