pipeline {

    agent any

    parameters {
        choice(name: 'TEST_TYPE', choices: ['LOAD', 'STRESS', 'SPIKE'], description: 'Select Test Type')
        choice(name: 'ENV', choices: ['DEV', 'QA', 'PROD'], description: 'Select Environment')

        string(name: 'THREAD_GROUPS', defaultValue: 'FishFlow', description: 'FishFlow,DogFlow,CatFlow')

        // NEW (runtime control)
        string(name: 'THREAD_CONFIG', defaultValue: 'FishFlow:5', description: 'FishFlow:10,DogFlow:20')
        string(name: 'RAMPUP', defaultValue: '', description: 'Optional override ramp-up')
        string(name: 'DURATION', defaultValue: '', description: 'Optional override duration')
    }

    environment {
        PT_REPO = 'https://github.com/ManishaS1713/MultiThreadGroupsInJmx_Jenkins.git'
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
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

                    echo "Thread Groups: ${env.SELECTED_TG}"
                    echo "Thread Config: ${env.THREAD_MAP}"
                }
            }
        }

        stage('Set Environment URL') {
            steps {
                script {
                    if (params.ENV == 'DEV') {
                        env.HOST = "petstore.octoperf.com"
                    } else if (params.ENV == 'QA') {
                        env.HOST = "petstore.octoperf.com"
                    } else {
                        env.HOST = "petstore.octoperf.com"
                    }

                    echo "Environment: ${params.ENV}"
                    echo "Host: ${env.HOST}"
                }
            }
        }

        stage('Set Test Type Configuration') {
            steps {
                script {
                    // Default values from TEST_TYPE
                    if (params.TEST_TYPE == 'LOAD') {
                        env.RAMPUP_VAL = "2"
                        env.DURATION_VAL = "60"
                    } else if (params.TEST_TYPE == 'STRESS') {
                        env.RAMPUP_VAL = "5"
                        env.DURATION_VAL = "60"
                    } else {
                        env.RAMPUP_VAL = "1"
                        env.DURATION_VAL = "60"
                    }

                    // 🔥 Override if user gives runtime values
                    if (params.RAMPUP?.trim()) {
                        env.RAMPUP_VAL = params.RAMPUP
                    }

                    if (params.DURATION?.trim()) {
                        env.DURATION_VAL = params.DURATION
                    }

                    echo "Final Ramp-up: ${env.RAMPUP_VAL}"
                    echo "Final Duration: ${env.DURATION_VAL}"
                }
            }
        }

        stage('Run Performance Test') {
            steps {
                bat """
                IF EXIST MultiThreadGroupsInJmx_Jenkins-result.jtl del MultiThreadGroupsInJmx_Jenkins-result.jtl
                IF EXIST MultiThreadGroupsInJmx_Jenkins-report rmdir /s /q MultiThreadGroupsInJmx_Jenkins-report

                C:\\Jmeter\\apache-jmeter-5.6.3\\bin\\jmeter.bat -n ^
                -t MultiThreadGroupsInJmx_Jenkins.jmx ^
                -Jhost=${env.HOST} ^
                -JthreadGroups=${env.SELECTED_TG} ^
                -JthreadConfig=${env.THREAD_MAP} ^
                -Jrampup=${env.RAMPUP_VAL} ^
                -Jduration=${env.DURATION_VAL} ^
                -l MultiThreadGroupsInJmx_Jenkins-result.jtl ^
                -e -o MultiThreadGroupsInJmx_Jenkins-report
                """
            }
        }

        stage('Generate Summary') {
            steps {
                script {

                    def raw = powershell(
                        returnStdout: true,
                        script: '''
                        $data = Import-Csv "MultiThreadGroupsInJmx_Jenkins-result.jtl"

                        if ($data.Count -eq 0) {
                            Write-Output "TOTAL=0"
                            Write-Output "SUCCESS=0"
                            Write-Output "FAIL=0"
                            Write-Output "AVG=0"
                            Write-Output "TPS=0"
                            Write-Output "ERRORPCT=0"
                            exit
                        }

                        $total = $data.Count
                        $success = ($data | Where-Object {$_.success -eq "true"}).Count
                        $fail = $total - $success
                        $avg = [math]::Round(($data | Measure-Object -Property elapsed -Average).Average,2)

                        $start = $data[0].timeStamp
                        $end = $data[-1].timeStamp
                        $duration = ($end - $start)/1000
                        if ($duration -eq 0) { $duration = 1 }

                        $tps = [math]::Round($total / $duration,2)
                        $errorPct = [math]::Round(($fail/$total)*100,2)

                        Write-Output "TOTAL=$total"
                        Write-Output "SUCCESS=$success"
                        Write-Output "FAIL=$fail"
                        Write-Output "AVG=$avg"
                        Write-Output "TPS=$tps"
                        Write-Output "ERRORPCT=$errorPct"
                        '''
                    ).trim()

                    echo raw

                    def lines = raw.split("\\r?\\n")

                    env.TOTAL = lines.find { it.startsWith("TOTAL=") }?.split("=")[1]
                    env.SUCCESS = lines.find { it.startsWith("SUCCESS=") }?.split("=")[1]
                    env.FAIL = lines.find { it.startsWith("FAIL=") }?.split("=")[1]
                    env.AVG = lines.find { it.startsWith("AVG=") }?.split("=")[1]
                    env.TPS = lines.find { it.startsWith("TPS=") }?.split("=")[1]
                    env.ERRORPCT = lines.find { it.startsWith("ERRORPCT=") }?.split("=")[1]

                    env.SLA_STATUS = "PASS"
                    if ((env.AVG ?: "0").toFloat() > 1000 || (env.ERRORPCT ?: "0").toFloat() > 1) {
                        env.SLA_STATUS = "FAIL"
                    }
                }
            }
        }

        stage('Publish Report') {
            steps {
                publishHTML([
                    allowMissing: true,
                    reportDir: 'MultiThreadGroupsInJmx_Jenkins-report',
                    reportFiles: 'index.html',
                    reportName: 'JMeter Report',
                    keepAll: true,
                    alwaysLinkToLastBuild: true
                ])
            }
        }

        stage('Send Email') {
            steps {
                script {
                    emailext(
                        subject: "Performance Test Result - ${env.SLA_STATUS}",
                        body: """
<h2>Performance Test Report (${params.ENV})</h2>

<b>Thread Groups:</b> ${env.SELECTED_TG} <br>
<b>Thread Config:</b> ${env.THREAD_MAP} <br>
<b>Ramp-up:</b> ${env.RAMPUP_VAL} <br>
<b>Duration:</b> ${env.DURATION_VAL} <br>

<b>Total Requests:</b> ${env.TOTAL} <br>
<b>Success:</b> ${env.SUCCESS} <br>
<b>Failures:</b> ${env.FAIL} <br>
<b>Avg Response Time:</b> ${env.AVG} ms <br>
<b>Throughput:</b> ${env.TPS} req/sec <br>
<b>Error %:</b> ${env.ERRORPCT} % <br>

<br><b>SLA Status:</b> <span style="color:${env.SLA_STATUS == 'PASS' ? 'green' : 'red'}">${env.SLA_STATUS}</span>

<br><br>
<a href="${BUILD_URL}">👉 View Full Report</a>
""",
                        to: 'manishas@ivavsys.com',
                        mimeType: 'text/html'
                    )
                }
            }
        }
    }
}
