pipeline {
    agent any

    environment {
        ACE_HOME          = 'C:\\Program Files\\IBM\\ACE\\13.0.8.0'
        ACE_HOST          = 'localhost'
        ACE_PORT          = '7601'
        EXPECTED_ACE_PORT = '7600'
        BAR_NAME          = 'ACE_Jenkins.bar'
        ACE_PROJECT       = 'ACE_Jenkins'
        ACE_SERVER_NAME   = 'ACE_Jenkins_Server'
    }

    stages {

        stage('Test Pipeline') {
            steps {
                echo '================================'
                echo 'ACE CI/CD Pipeline Started'
                echo '================================'
            }
        }

        stage('Check ACE Environment') {
            steps {
                bat '''
                call "%ACE_HOME%\\server\\bin\\mqsiprofile.cmd"
                echo ACE Environment Loaded
                where ibmint
                ibmint --version
                '''
            }
        }

        stage('Build BAR and Analyze Errors') {
            steps {
                script {

                    bat '''
                    if not exist build mkdir build
                    if exist "build\\build.log" del "build\\build.log"
                    '''

                    def buildStatus = bat(
                        returnStatus: true,
                        script: '''
                        call "%ACE_HOME%\\server\\bin\\mqsiprofile.cmd"

                        ibmint package ^
                          --input-path "%WORKSPACE%" ^
                          --output-bar-file "%WORKSPACE%\\build\\%BAR_NAME%" ^
                          --project %ACE_PROJECT% ^
                          > "%WORKSPACE%\\build\\build.log" 2>&1

                        exit /b %ERRORLEVEL%
                        '''
                    )

                    echo '================================'
                    echo 'ACE BAR BUILD LOG'
                    echo '================================'

                    bat '''
                    type "%WORKSPACE%\\build\\build.log"
                    '''

                    if (buildStatus != 0) {

                        def buildLog = readFile(
                            file: "${env.WORKSPACE}\\build\\build.log"
                        )

                        echo '================================'
                        echo 'ACE BAR BUILD FAILED'
                        echo 'AUTOMATIC ERROR ANALYSIS'
                        echo '================================'

                        printBipMessages(buildLog)
                        analyzeBipKnowledgeBase(buildLog)

                        if (buildLog.toLowerCase().contains('project') ||
                            buildLog.toLowerCase().contains('compile')) {

                            echo 'ERROR CATEGORY : ACE Build / Compilation Error'
                            echo 'POSSIBLE REASON : ACE project contains build or compilation errors.'
                            echo 'CHECK 1 : Message Flow, ESQL and project resources.'
                            echo 'FIX     : Correct the flow/ESQL compile error shown in build.log.'
                            echo 'CHECK 2 : Project name and workspace structure.'
                            echo "FIX     : Verify ACE_PROJECT = '${env.ACE_PROJECT}' and repository folder structure."
                            echo 'SUGGESTED FIX : Resolve the first BIP error reported in build.log.'

                        } else {

                            echo 'ERROR CATEGORY : Unknown ACE Build Error'
                            echo 'SUGGESTED FIX : Review BIP messages in build.log.'
                        }

                        error('ACE BAR build failed. Automatic error analysis completed.')

                    } else {

                        echo '================================'
                        echo 'ACE BAR BUILD SUCCESSFUL'
                        echo '================================'

                        bat '''
                        dir "%WORKSPACE%\\build\\%BAR_NAME%"
                        '''
                    }
                }
            }
        }

        stage('Deploy BAR and Analyze Errors') {
            steps {
                script {

                    bat '''
                    if exist "build\\deploy.log" del "build\\deploy.log"
                    '''

                    def deployStatus = bat(
                        returnStatus: true,
                        script: '''
                        call "%ACE_HOME%\\server\\bin\\mqsiprofile.cmd"

                        ibmint deploy ^
                          --input-bar-file "%WORKSPACE%\\build\\%BAR_NAME%" ^
                          --output-host %ACE_HOST% ^
                          --output-port %ACE_PORT% ^
                          --https ^
                          --insecure ^
                          > "%WORKSPACE%\\build\\deploy.log" 2>&1

                        exit /b %ERRORLEVEL%
                        '''
                    )

                    echo '================================'
                    echo 'ACE DEPLOYMENT LOG'
                    echo '================================'

                    bat '''
                    type "%WORKSPACE%\\build\\deploy.log"
                    '''

                    if (deployStatus != 0) {

                        def deployLog = readFile(
                            file: "${env.WORKSPACE}\\build\\deploy.log"
                        )

                        echo '================================'
                        echo 'ACE DEPLOYMENT FAILED'
                        echo 'STARTING AUTOMATIC ERROR ANALYSIS'
                        echo '================================'

                        printBipMessages(deployLog)

                        echo '================================'
                        echo 'BIP KNOWLEDGE BASE ANALYSIS'
                        echo '================================'

                        analyzeBipKnowledgeBase(deployLog)

                        if (deployLog.contains('BIP1921S') ||
                            deployLog.contains('BIP8032E') ||
                            deployLog.toLowerCase().contains('connection refused')) {

                            echo '================================'
                            echo 'ERROR CATEGORY : ACE Connectivity / Admin REST Error'
                            echo "TARGET HOST    : ${env.ACE_HOST}"
                            echo "TARGET PORT    : ${env.ACE_PORT}"
                            echo '================================'

                            echo 'CHECK 1 : Integration Server is running.'
                            echo 'IF FAILED:'
                            echo "Start Integration Server '${env.ACE_SERVER_NAME}' and rerun the pipeline."
                            echo ''

                            echo "CHECK 2 : Admin REST port ${env.ACE_PORT} is correct and listening."
                            echo 'IF FAILED:'

                            if (env.ACE_PORT != env.EXPECTED_ACE_PORT) {
                                echo 'Detected configuration mismatch.'
                                echo "FROM : ACE_PORT = '${env.ACE_PORT}'"
                                echo "TO   : ACE_PORT = '${env.EXPECTED_ACE_PORT}'"
                                echo 'Change the Jenkinsfile value, commit it, and rerun the pipeline.'
                            } else {
                                echo "ACE_PORT is already set to ${env.ACE_PORT}."
                                echo "Verify that port ${env.ACE_PORT} is actually listening on the ACE server."
                                echo "Example command: netstat -ano | findstr :${env.ACE_PORT}"
                            }

                            echo ''

                            echo 'CHECK 3 : Host name or IP address is correct.'
                            echo 'IF FAILED:'
                            echo 'Change ACE_HOST in Jenkinsfile to the real ACE server host name or IP address.'
                            echo "Current value : ACE_HOST = '${env.ACE_HOST}'"
                            echo "For this local lab, expected value is: ACE_HOST = 'localhost'"
                            echo ''

                            echo 'CHECK 4 : Jenkins machine can access the ACE server.'
                            echo 'IF FAILED:'
                            echo 'Verify network connectivity, DNS/IP resolution and firewall rules.'
                            echo "Allow Jenkins machine to connect to ${env.ACE_HOST}:${env.ACE_PORT}."
                            echo 'For a remote ACE server, confirm the Admin REST port is open between Jenkins and ACE.'
                            echo ''

                            echo 'CHECK 5 : HTTP/HTTPS matches ACE Admin SSL configuration.'
                            echo 'IF FAILED:'
                            echo 'If ACE Admin SSL is enabled, use --https.'
                            echo 'If ACE Admin SSL is disabled, use --no-https.'
                            echo 'If certificate validation fails, trust the ACE admin certificate.'
                            echo 'Use --insecure only for local/test environments.'
                            echo ''

                            echo '================================'
                            echo 'EXACT REMEDIATION SUMMARY'
                            echo '================================'

                            if (env.ACE_PORT != env.EXPECTED_ACE_PORT) {
                                echo "PRIMARY FIX : Change ACE_PORT from '${env.ACE_PORT}' to '${env.EXPECTED_ACE_PORT}'."
                            } else {
                                echo 'PRIMARY FIX : Port value matches expected configuration.'
                                echo 'Next verify server status, listening port, host reachability and HTTPS settings.'
                            }

                        } else if (deployLog.contains('BIP3165E') ||
                                   deployLog.toLowerCase().contains('ssl') ||
                                   deployLog.toLowerCase().contains('certificate')) {

                            echo '================================'
                            echo 'ERROR CATEGORY : ACE SSL / HTTPS Error'
                            echo "TARGET HOST    : ${env.ACE_HOST}"
                            echo "TARGET PORT    : ${env.ACE_PORT}"
                            echo '================================'

                            echo 'CHECK 1 : ACE Admin SSL setting.'
                            echo 'IF FAILED:'
                            echo 'Run: ibmint display admin-ssl'
                            echo ''

                            echo 'CHECK 2 : Jenkins deploy protocol.'
                            echo 'IF FAILED:'
                            echo 'Admin SSL enabled  -> use --https'
                            echo 'Admin SSL disabled -> use --no-https'
                            echo ''

                            echo 'CHECK 3 : Certificate trust.'
                            echo 'IF FAILED:'
                            echo 'Import/trust the ACE admin SSL certificate on the Jenkins machine.'
                            echo 'For this local lab, --insecure can be used temporarily.'
                            echo ''

                            echo 'SUGGESTED FIX : Align Jenkins deploy protocol with ACE Admin SSL configuration.'

                        } else if (deployLog.contains('BIP8081E') ||
                                   deployLog.toLowerCase().contains('deploy') ||
                                   deployLog.toLowerCase().contains('application')) {

                            echo '================================'
                            echo 'ERROR CATEGORY : ACE Application Deployment / Processing Error'
                            echo '================================'

                            echo 'CHECK 1 : BAR file contents.'
                            echo 'IF FAILED:'
                            echo 'Rebuild the BAR and verify required application resources are included.'
                            echo ''

                            echo 'CHECK 2 : Application dependencies.'
                            echo 'IF FAILED:'
                            echo 'Add or fix missing libraries, policies, schemas or referenced resources.'
                            echo ''

                            echo 'CHECK 3 : Application configuration.'
                            echo 'IF FAILED:'
                            echo 'Correct deployment overrides or environment-specific configuration.'
                            echo ''

                            echo 'CHECK 4 : Earlier BIP messages.'
                            echo 'IF FAILED:'
                            echo 'Fix the first specific BIP error before BIP8081E.'
                            echo ''

                            echo 'SUGGESTED FIX : Resolve the earliest concrete BIP error in deploy.log.'

                        } else {

                            echo '================================'
                            echo 'ERROR CATEGORY : Unknown ACE Deployment Error'
                            echo '================================'

                            echo 'POSSIBLE REASON : Error does not match a known automation rule.'
                            echo 'SUGGESTED FIX : Review detected BIP messages in deploy.log and extend the knowledge base.'
                        }

                        error('ACE BAR deployment failed. Automatic error analysis completed.')

                    } else {

                        echo '================================'
                        echo 'ACE BAR DEPLOYMENT SUCCESSFUL'
                        echo "SERVER : ${env.ACE_HOST}:${env.ACE_PORT}"
                        echo '================================'
                    }
                }
            }
        }
    }

    post {

        success {
            echo '================================'
            echo 'ACE CI/CD PIPELINE SUCCESS'
            echo '================================'
        }

        failure {
            echo '================================'
            echo 'ACE CI/CD PIPELINE FAILED'
            echo 'CHECK AUTOMATIC ERROR ANALYSIS ABOVE'
            echo '================================'
        }

        always {
            archiveArtifacts(
                artifacts: 'build/*.log,build/*.bar',
                allowEmptyArchive: true
            )
        }
    }
}


/*
 * Print all ACE BIP messages
 */
def printBipMessages(String logText) {

    echo '================================'
    echo 'DETECTED BIP MESSAGES'
    echo '================================'

    def lines = logText.split('\\r?\\n')
    def found = false

    for (int i = 0; i < lines.length; i++) {

        def currentLine = lines[i].trim()

        if (currentLine ==~ /.*BIP[0-9]+[A-Z]:.*/) {
            echo currentLine
            found = true
        }
    }

    if (!found) {
        echo 'No BIP error code detected.'
    }

    echo '================================'
}


/*
 * ACE BIP Knowledge Base
 */
def analyzeBipKnowledgeBase(String logText) {

    def foundKnownError = false

    if (logText.contains('BIP1921S')) {

        foundKnownError = true

        echo '--------------------------------'
        echo 'BIP CODE : BIP1921S'
        echo 'TYPE     : Server Connectivity'
        echo 'MEANING  : ACE Integration Server cannot be reached.'
        echo 'CAUSE    : Server stopped, incorrect host/port, or protocol mismatch.'
        echo 'ACTION   : Check server status, host, Admin REST port and protocol.'
        echo '--------------------------------'
    }

    if (logText.contains('BIP3165E')) {

        foundKnownError = true

        echo '--------------------------------'
        echo 'BIP CODE : BIP3165E'
        echo 'TYPE     : SSL / Socket Communication'
        echo 'MEANING  : An SSL socket operation failed.'
        echo 'CAUSE    : Connection refused, SSL configuration, or certificate issue.'
        echo 'ACTION   : Verify port, HTTPS setting, Admin SSL and certificate.'
        echo '--------------------------------'
    }

    if (logText.contains('BIP8032E')) {

        foundKnownError = true

        echo '--------------------------------'
        echo 'BIP CODE : BIP8032E'
        echo 'TYPE     : Administration Connection'
        echo 'MEANING  : Unable to connect to ACE Integration Server or Integration Node.'
        echo 'CAUSE    : Admin REST endpoint is unavailable or connection details are incorrect.'
        echo 'ACTION   : Verify host, Admin REST port, server status and protocol.'
        echo '--------------------------------'
    }

    if (logText.contains('BIP8081E')) {

        foundKnownError = true

        echo '--------------------------------'
        echo 'BIP CODE : BIP8081E'
        echo 'TYPE     : Command Processing'
        echo 'MEANING  : ACE command processing failed.'
        echo 'CAUSE    : Usually a higher-level error caused by an earlier BIP message.'
        echo 'ACTION   : Find and resolve the first specific BIP error before BIP8081E.'
        echo '--------------------------------'
    }

    if (!foundKnownError) {

        echo '--------------------------------'
        echo 'KNOWLEDGE BASE RESULT'
        echo 'No known BIP mapping found.'
        echo 'ACTION : Review the detected BIP messages and extend the knowledge base.'
        echo '--------------------------------'
    }
}
