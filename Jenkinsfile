pipeline {

    agent any

    parameters {
        choice(name: 'TEST_TYPE', choices: ['LOAD', 'STRESS', 'SPIKE'], description: 'Test Type')
        choice(name: 'ENV', choices: ['DEV', 'QA', 'PROD'], description: 'Environment')

        string(name: 'THREAD_GROUPS', defaultValue: 'FishFlow', description: 'Comma separated: FishFlow,CatFlow')

        string(name: 'FISH_THREADS', defaultValue: '', description: 'Threads for FishFlow')
        string(name: 'CAT_THREADS', defaultValue: '', description: 'Threads for CatFlow')

        string(name: 'RAMPUP', defaultValue: '2', description: 'Ramp-up time')
        string(name: 'DURATION', defaultValue: '60', description: 'Test duration')
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

        stage('Prepare Thread Values') {
            steps {
                script {

                    def selected = params.THREAD_GROUPS.replaceAll("\\s","").split(",")

                    // Dynamic assignment
                    env.FISH = selected.contains("FishFlow") ? (params.FISH_THREADS?.trim() ?: "1") : "0"
                    env.CAT  = selected.contains("CatFlow")  ? (params.CAT_THREADS?.trim()  ?: "1") : "0"

                    echo "Selected Flows: ${params.THREAD_GROUPS}"
                    echo "Fish Threads: ${env.FISH}"
                    echo "Cat Threads: ${env.CAT}"
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
                IF EXIST MultiThreadGroupsInJmx_Jenkins-result.jtl del MultiThreadGroupsInJmx_Jenkins-result.jtl
                IF EXIST MultiThreadGroupsInJmx_Jenkins-report rmdir /s /q MultiThreadGroupsInJmx_Jenkins-report

                C:\\Jmeter\\apache-jmeter-5.6.3\\bin\\jmeter.bat -n ^
                -t MultiThreadGroupsInJmx_Jenkins.jmx ^
                -Jhost=${env.HOST} ^
                -JFishFlow_threads=${env.FISH} ^
                -JCatFlow_threads=${env.CAT} ^
                -Jrampup=${params.RAMPUP} ^
                -Jduration=${params.DURATION} ^
                -l MultiThreadGroupsInJmx_Jenkins-result.jtl ^
                -e -o MultiThreadGroupsInJmx_Jenkins-report
                """
            }
        }

        stage('Generate Summary (P90)') {
            steps {
                script {

                    def raw = powershell(
                        returnStdout: true,
                        script: '''
                        if (!(Test-Path "MultiThreadGroupsInJmx_Jenkins-result.jtl")) {
                            Write-Output "TOTAL=0"
                            Write-Output "P90=0"
                            Write-Output "ERRORPCT=0"
                            exit
                        }

                        $data = Get-Content "MultiThreadGroupsInJmx_Jenkins-result.jtl" | ConvertFrom-Csv

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
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: 'MultiThreadGroupsInJmx_Jenkins-report',
                    reportFiles: 'index.html',
                    reportName: 'JMeter Report'
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

<b>Flows:</b> ${params.THREAD_GROUPS} <br>
<b>Fish Threads:</b> ${env.FISH} <br>
<b>Cat Threads:</b> ${env.CAT} <br>

<b>Total:</b> ${env.TOTAL} <br>
<b>P90:</b> ${env.P90} ms <br>
<b>Error %:</b> ${env.ERRORPCT}% <br>

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
