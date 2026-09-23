pipeline {
    agent any
    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'test', 'prod'],
            description: 'Select target Layer7 environment'
        )
        string(
            name: 'RELEASE',
            defaultValue: 'R1',
            description: 'Release to run: R1, R2, R3, R4, or ALL. Case-insensitive.'
        )
        string(
            name: 'APP_FILTER',
            defaultValue: '',
            description: 'Optional: comma/space-separated app names to limit within the release (e.g. "HRT-API,CUSTOMER-API"). Blank = all apps in the release.'
        )
    }
    environment {
        GMU_HOME = 'C:\\gmu'
        JAVA_HOME = 'C:\\Program Files\\Java\\jdk-17.0.18'
        PATH = "${env.JAVA_HOME}\\bin;${env.PATH}"
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Load Environment & Manifest Config') {
            steps {
                script {
                    // Load basic Environment configurations
                    def envConfig = readYaml file: "config/${params.ENVIRONMENT}.yaml"
                    env.GATEWAY_HOST = envConfig.gateway.host
                    env.GATEWAY_PORT = envConfig.gateway.port.toString()
                    env.GATEWAY_PROTOCOL = envConfig.gateway.protocol
                    echo "Target Environment : ${params.ENVIRONMENT}"
                    echo "Gateway Host       : ${env.GATEWAY_HOST}"
                    echo "Gateway Port       : ${env.GATEWAY_PORT}"
 
                    // ── Determine releases ────────────────────────────────────────
                    def releaseInput = params.RELEASE?.trim()?.toUpperCase() ?: 'R1'
                    def releasesToRun = []

                    if (releaseInput == 'ALL') {
                        releasesToRun = ['R1', 'R2', 'R3', 'R4'].findAll { fileExists("releases/${it}/manifest.yaml") }
                    } else {
                        // Splits by commas or spaces, trims whitespace, and ensures manifest exists
                        releasesToRun = releaseInput.split('[,\\s]+')
                                            .collect { it.trim() }
                                            .findAll { it && fileExists("releases/${it}/manifest.yaml") }
                    }
 
                    // ── Optional app-level filter ──────────────────────────────
                    def appFilter = params.APP_FILTER?.trim()
                        ? params.APP_FILTER.split('[,\\s]+').collect { it.trim() }.findAll { it }
                        : []
 
                    // ── Map tracking database generation ────────────────────────
                    // Tracks releases to their apis list: ['R1': ['CUSTOMER-API']]
                    logReleaseApiMap = [:] 
                    releasesToRun.each { release ->
                        def manifestPath = "releases/${release}/manifest.yaml"
                        if (!fileExists(manifestPath)) {
                            error "Release ${release}: manifest not found at ${manifestPath}"
                        }
                        def manifest = readYaml file: manifestPath
                        // Parse the services array from your manifest layout
                        def apps = (manifest.services ?: []).collect { it.toString() }
 
                        // If user defined an APP_FILTER, restrict the execution pool
                        if (appFilter) {
                            def filterLower = appFilter.collect { it.toLowerCase() }
                            apps = apps.findAll { filterLower.contains(it.toLowerCase()) }
                            echo "Release ${release}: filtered to [${apps.join(', ')}]"
                        }
                       if (apps.isEmpty()) {
                            error "No valid APIs found to process for Release ${release} with current APP_FILTER."
                        }
 
                        logReleaseApiMap[release] = apps
                        echo "Final apps to process for Release ${release}: [${apps.join(', ')}]"
                    }
                }
            }
        }
    stage('Validate GMU Directory Structure') {
    steps {
        script {
            logReleaseApiMap.each { release, apps ->
                apps.each { app ->

                    String apiDir = "apis/${app}"

                    if (!fileExists(apiDir)) {
                        error "API directory not found: ${apiDir}"
                    }

                    if (!fileExists("${apiDir}/mappings.xml")) {
                        error "mappings.xml missing in ${apiDir}"
                    }

                    if (!fileExists("${apiDir}/dependencies.xml")) {
                        error "dependencies.xml missing in ${apiDir}"
                    }

                    if (!fileExists("${apiDir}/rootFolder")) {
                        error "rootFolder directory missing in ${apiDir}"
                    }

                    if (!fileExists("${apiDir}/dependencies")) {
                        error "dependencies directory missing in ${apiDir}"
                    }

                    def rootFiles = findFiles(glob: "${apiDir}/rootFolder/**/*")

                    if (rootFiles.length == 0) {
                        error "rootFolder is empty for ${app}"
                    }

                    echo "Validated GMU format directory structure for ${app}"
                    }
                }
            }
        }
    }
        stage('Validate GMU') {
            steps {
                bat '''
                    @echo off
                    echo Checking GMU installation...
                    if not exist "%GMU_HOME%\\GatewayMigrationUtility.bat" (
                        echo GMU not found at %GMU_HOME%
                        exit /b 1
                    )
                    echo GMU installation found.
                '''
            }
        }
 
        stage('Test Layer7 Connectivity') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'layer7-gateway-credentials',
                        usernameVariable: 'GATEWAY_USERNAME',
                        passwordVariable: 'GATEWAY_PASSWORD'
                    )
                ]) {
                    script {
                        logReleaseApiMap.each { release, apps ->
                            apps.each { app ->
                                // Pointing to apis/ folder
                                String currentBundlePath = "apis/${app}"
                                echo "========================================"
                                echo "Testing connectivity using api folder: ${currentBundlePath}"
                                echo "========================================"
                                bat """
                                    set "PATH=%JAVA_HOME%\\bin;%PATH%"
                                    "%GMU_HOME%\\GatewayMigrationUtility.bat" migrateIn ^
                                        -h "%GATEWAY_HOST%" ^
                                        -p "%GATEWAY_PORT%" ^
                                        -u "%GATEWAY_USERNAME%" ^
                                        --plaintextPassword "%GATEWAY_PASSWORD%" ^
                                        --bundle "${currentBundlePath}" ^
                                        --plaintextEncryptionPassphrase "%GATEWAY_PASSWORD%" ^
                                        --results "results-${app}.xml" ^
                                        --action NewOrUpdate ^
                                        --trustCertificate ^
                                        --trustHostname ^
                                        --test
                                """
                            }
                        }
                    }
                }
            }
        }
        stage('Deploy to Layer7') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'layer7-gateway-credentials',
                        usernameVariable: 'GATEWAY_USERNAME',
                        passwordVariable: 'GATEWAY_PASSWORD'
                    )
                ]) {
                    script {
                        logReleaseApiMap.each { release, apps ->
                            apps.each { app ->
                                // Pointing to apis/ folder
                                String currentBundlePath = "apis/${app}"
                                echo "Deploying API: ${currentBundlePath} to Layer7 Gateway..."
                                int exitCode = bat(
                                    returnStatus: true,
                                    script: """
                                        @echo off
                                        call "%GMU_HOME%\\GatewayMigrationUtility.bat" ^
                                            migrateIn ^
                                            --host %GATEWAY_HOST% ^
                                            --port %GATEWAY_PORT% ^
                                            --username "%GATEWAY_USERNAME%" ^
                                            --plaintextPassword "%GATEWAY_PASSWORD%" ^
                                            --bundle "${currentBundlePath}" ^
                                            --plaintextEncryptionPassphrase "%GATEWAY_PASSWORD%" ^
                                            --results "gmu-results-${app}.xml" ^
                                            --action NewOrUpdate ^
                                            --trustCertificate ^
                                            --trustHostname
                                    """
                                )
                                if (exitCode != 0) {
                                    error "Deployment failed for API ${app} with error code ${exitCode}."
                                }
                                echo "Successfully deployed: ${app}"
                            }
                        }
                    }
                }
            }
        }
        stage('Deployment Verification') {
            steps {
                echo "All specified GMU deployments completed successfully."
            }
        }
    }
    post {
        success {
            script {
                echo """
                ==========================================
                Layer7 Deployment SUCCESS
                Environment : ${params.ENVIRONMENT}
                Processed Deployments:
                """
                logReleaseApiMap.each { release, apps ->
                    echo "Release ${release}: ${apps.join(', ')}"
                }
                echo "=========================================="
            }
        }
        failure {
            echo """
            ==========================================
            Layer7 Deployment FAILED
            Environment : ${params.ENVIRONMENT}
            ==========================================
            """
        }
        always {
        archiveArtifacts artifacts: 'results-*.xml, gmu-results-*.xml', allowEmptyArchive: true
        cleanWs()
    }
    }
}
