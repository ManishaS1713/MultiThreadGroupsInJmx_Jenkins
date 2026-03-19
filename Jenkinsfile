pipeline {

    agent any

    parameters {
        choice(name: 'TEST_TYPE', choices: ['LOAD', 'STRESS', 'SPIKE'], description: 'Select Test Type')
        choice(name: 'ENV', choices: ['DEV', 'QA', 'PROD'], description: 'Select Environment')

        string(name: 'THREAD_GROUPS', defaultValue: 'FishFlow', description: 'FishFlow,DogFlow,CatFlow')

        string(name: 'FISH_THREADS', defaultValue: '5', description: 'FishFlow threads')
        string(name: 'CAT_THREADS', defaultValue: '5', description: 'CatFlow threads')

        string(name: 'RAMPUP', defaultValue: '2', description: 'Ramp-up')
        string(name: 'DURATION', defaultValue: '60', description: 'Duration')
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
                    env.SELECTED_TG = params.THREAD_GROUPS.replaceAll("\\s","")

                    echo "Thread Groups: ${env.SELECTED_TG}"
                    echo "Fish Threads: ${params.FISH_THREADS}"
                    echo "Cat Threads: ${params.CAT_THREADS}"
                }
            }
        }

        stage('Set Environment URL') {
            steps {
                script {
                    env.HOST = "petstore.octoperf.com"
                    echo "Environment: ${params.ENV}"
                    echo "Host: ${env.HOST}"
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
                -Jhost=${env.HOST} ^
                -JthreadGroups=${env.SELECTED_TG} ^
                -JFishFlow_threads=${params.FISH_THREADS} ^
                -JCatFlow_threads=${params.CAT_THREADS} ^
                -Jrampup=${params.RAMPUP} ^
                -Jduration=${params.DURATION} ^
                -l result.jtl ^
                -e -o report
                """
            }
        }

        stage('Generate Summary (FAST P90)') {
            steps {
                script {

                    def raw = powershell(
                        returnStdout: true,
                        script: '''
                        if (!(Test-Path "result.jtl")) {
                            Write-Output "TOTAL=0"
                            Write-Output "P90=0"
                            Write-Output "ERRORPCT=0"
                            exit
                        }

                        # Read only last 10000 records (avoid memory issue)
                        $lines = Get-Content "result.jtl" -Tail 10000
                        $data = $lines | ConvertFrom-Csv

                        if ($data.Count -eq 0) {
                            Write-Output "TOTAL=0"
                            Write-Output "P90=0"
                            Write-Output "ERRORPCT=0"
                            exit
                        }

                        $total = $data.Count
                        $fail = ($data | Where-Object {$_.success -ne "true"}).Count
                        $errorPct = [math]::Round(($fail/$total)*100,2)

                        $sorted = $data | Sort-Object {[int]$_.elapsed}
                        $index = [math]::Ceiling(0.9 * $sorted.Count) - 1
                        $p90 = $sorted[$index].elapsed

                        Write-Output "TOTAL=$total"
                        Write-Output "P90=$p90"
                        Write-Output "ERRORPCT=$errorPct"
                        '''
                    ).trim()

                    echo raw

                    def lines = raw.split("\\r?\\n")

                    env.TOTAL = lines.find { it.startsWith("TOTAL=") }?.split("=")[1]
                    env.P90 = lines.find { it.startsWith("P90=") }?.split("=")[1]
                    env.ERRORPCT = lines.find { it.startsWith("ERRORPCT=") }?.split("=")[1]

                    // SLA
                    env.SLA_STATUS = (env.P90.toFloat() <= 1500 && env.ERRORPCT.toFloat() <= 1) ? "PASS" : "FAIL"

                    if (env.SLA_STATUS == "FAIL") {
                        error "❌ SLA Failed"
                    }
                }
            }
        }

        stage('Publish Report') {
            steps {
                publishHTML([
                    allowMissing: true,
                    reportDir: 'report',
                    reportFiles: 'index.html',
                    reportName: 'JMeter Report',
                    keepAll: true
                ])
            }
        }

        stage('Send Email') {
            steps {
                script {
                    emailext(
                        subject: "Performance Test - ${env.SLA_STATUS}",
                        body: """
<h3>Performance Report</h3>

Total: ${env.TOTAL} <br>
P90: ${env.P90} ms <br>
Error %: ${env.ERRORPCT}% <br>

<b>Status:</b> ${env.SLA_STATUS}

<br><br>
<a href="${BUILD_URL}">View Report</a>
""",
                        to: 'manishas@ivavsys.com',
                        mimeType: 'text/html'
                    )
                }
            }
        }
    }
}
