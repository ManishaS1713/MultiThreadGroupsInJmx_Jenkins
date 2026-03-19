pipeline {

    agent any

    parameters {
        choice(name: 'TEST_TYPE', choices: ['LOAD', 'STRESS', 'SPIKE'], description: 'Test Type')
        choice(name: 'ENV', choices: ['DEV', 'QA', 'PROD'], description: 'Environment')

        string(name: 'THREAD_GROUPS', defaultValue: 'FishFlow', description: 'FishFlow,DogFlow,CatFlow,BirdsFlow,ReptilesFlow')

        string(name: 'FISH_THREADS', defaultValue: '', description: 'Optional')
        string(name: 'DOG_THREADS', defaultValue: '', description: 'Optional')
        string(name: 'CAT_THREADS', defaultValue: '', description: 'Optional')
        string(name: 'BIRDS_THREADS', defaultValue: '', description: 'Optional')
        string(name: 'REPTILES_THREADS', defaultValue: '', description: 'Optional')

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

        stage('Build Thread Config') {
            steps {
                script {

                    env.SELECTED_TG = params.THREAD_GROUPS.replaceAll("\\s","")
                    def selected = env.SELECTED_TG.split(",")

                    def configList = []

                    selected.each { tg ->

                        if (tg == "FishFlow") {
                            def val = params.FISH_THREADS?.trim() ? params.FISH_THREADS : "1"
                            configList.add("FishFlow:${val}")
                        }

                        if (tg == "DogFlow") {
                            def val = params.DOG_THREADS?.trim() ? params.DOG_THREADS : "1"
                            configList.add("DogFlow:${val}")
                        }

                        if (tg == "CatFlow") {
                            def val = params.CAT_THREADS?.trim() ? params.CAT_THREADS : "1"
                            configList.add("CatFlow:${val}")
                        }

                        if (tg == "BirdsFlow") {
                            def val = params.BIRDS_THREADS?.trim() ? params.BIRDS_THREADS : "1"
                            configList.add("BirdsFlow:${val}")
                        }

                        if (tg == "ReptilesFlow") {
                            def val = params.REPTILES_THREADS?.trim() ? params.REPTILES_THREADS : "1"
                            configList.add("ReptilesFlow:${val}")
                        }
                    }

                    env.THREAD_CONFIG = configList.join(",")

                    echo "Selected Flows: ${env.SELECTED_TG}"
                    echo "Thread Config: ${env.THREAD_CONFIG}"
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
                -JthreadGroups=${env.SELECTED_TG} ^
                -JthreadConfig=${env.THREAD_CONFIG} ^
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

                        $lines = Get-Content "MultiThreadGroupsInJmx_Jenkins-result.jtl" -Tail 5000

                        # Remove duplicate headers
                        $clean = $lines | Select-Object -Unique

                        # Take last 10000 rows
                        $data = $clean | Select-Object -Last 10000 | ConvertFrom-Csv

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

<b>Flows:</b> ${env.SELECTED_TG} <br>
<b>Thread Config:</b> ${env.THREAD_CONFIG} <br>

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
