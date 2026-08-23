pipeline {
    agent any

    environment {
        ACE_HOME    = 'C:\\Program Files\\IBM\\ACE\\13.0.8.0'
        ACE_HOST    = 'localhost'
        ACE_PORT    = '7601'
        BAR_NAME    = 'ACE_Jenkins.bar'
        ACE_PROJECT = 'ACE_Jenkins'
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
                            echo 'CHECK 2 : Project name and workspace structure.'
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

                            echo 'POSSIBLE REASON : Integration Server cannot be reached.'
                            echo 'CHECK 1 : Integration Server is running.'
                            echo "CHECK 2 : Admin REST port ${env.ACE_PORT} is listening."
                            echo 'CHECK 3 : Host name or IP address is correct.'
                            echo 'CHECK 4 : Jenkins machine can access the ACE server.'
                            echo 'CHECK 5 : HTTP/HTTPS matches ACE Admin SSL configuration.'
                            echo "SUGGESTED FIX : Verify ${env.ACE_HOST}:${env.ACE_PORT} and Integration Server status."

                        } else if (deployLog.contains('BIP3165E') ||
                                   deployLog.toLowerCase().contains('ssl') ||
                                   deployLog.toLowerCase().contains('certificate')) {

                            echo '================================'
                            echo 'ERROR CATEGORY : ACE SSL / HTTPS Error'
                            echo "TARGET HOST    : ${env.ACE_HOST}"
                            echo "TARGET PORT    : ${env.ACE_PORT}"
                            echo '================================'

                            echo 'POSSIBLE REASON : Admin SSL protocol or certificate problem.'
                            echo 'CHECK 1 : ACE Admin SSL configuration.'
                            echo 'CHECK 2 : HTTPS is enabled when required.'
                            echo 'CHECK 3 : Certificate validity.'
                            echo 'CHECK 4 : Certificate trust configuration.'
                            echo 'SUGGESTED FIX : Verify Admin SSL and certificate configuration.'

                        } else if (deployLog.contains('BIP8081E') ||
                                   deployLog.toLowerCase().contains('deploy') ||
                                   deployLog.toLowerCase().contains('application')) {

                            echo '================================'
                            echo 'ERROR CATEGORY : ACE Application Deployment / Processing Error'
                            echo '================================'

                            echo 'POSSIBLE REASON : ACE failed to process the BAR deployment.'
                            echo 'CHECK 1 : BAR file contents.'
                            echo 'CHECK 2 : Application dependencies.'
                            echo 'CHECK 3 : Application configuration.'
                            echo 'CHECK 4 : Earlier BIP messages for actual root cause.'
                            echo 'SUGGESTED FIX : Resolve the first specific BIP error before BIP8081E.'

                        } else {

                            echo '================================'
                            echo 'ERROR CATEGORY : Unknown ACE Deployment Error'
                            echo '================================'

                            echo 'POSSIBLE REASON : Error does not match a known automation rule.'
                            echo 'SUGGESTED FIX : Review detected BIP messages in deploy.log.'
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
        echo 'ACTION   : Check Integration Server status and Admin REST connection.'
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
