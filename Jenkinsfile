pipeline {
    agent any

    environment {
        ACE_HOME   = 'C:\\Program Files\\IBM\\ACE\\13.0.8.0'
        ACE_HOST   = 'localhost'
        ACE_PORT   = '7600'
        BAR_NAME   = 'ACE_Jenkins.bar'
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

                        if (buildLog.toLowerCase().contains('project') ||
                            buildLog.toLowerCase().contains('compile')) {

                            echo 'ERROR CATEGORY : ACE Build / Compilation Error'
                            echo 'POSSIBLE REASON : ACE project contains build or compilation errors.'
                            echo 'CHECK : Message Flow, ESQL and project resources.'
                            echo 'CHECK : Project name and workspace structure.'
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

                        /*
                         * 1. CONNECTIVITY / ADMIN REST
                         */
                        if (deployLog.contains('BIP1921S') ||
                            deployLog.contains('BIP8032E') ||
                            deployLog.toLowerCase().contains('connection refused')) {

                            echo 'ERROR CATEGORY : ACE Connectivity / Admin REST Error'
                            echo "TARGET HOST    : ${env.ACE_HOST}"
                            echo "TARGET PORT    : ${env.ACE_PORT}"
                            echo 'POSSIBLE REASON : Integration Server cannot be reached.'
                            echo 'CHECK : Integration Server is running.'
                            echo "CHECK : Admin REST port ${env.ACE_PORT} is listening."
                            echo 'CHECK : Host name/IP address is correct.'
                            echo 'CHECK : Jenkins machine can access the server.'
                            echo "SUGGESTED FIX : Verify ${env.ACE_HOST}:${env.ACE_PORT} and Integration Server status."

                        /*
                         * 2. SSL / HTTPS
                         */
                        } else if (deployLog.contains('BIP3165E') ||
                                   deployLog.toLowerCase().contains('ssl') ||
                                   deployLog.toLowerCase().contains('certificate')) {

                            echo 'ERROR CATEGORY : ACE SSL / HTTPS Error'
                            echo "TARGET HOST    : ${env.ACE_HOST}"
                            echo "TARGET PORT    : ${env.ACE_PORT}"
                            echo 'POSSIBLE REASON : Admin SSL protocol or certificate problem.'
                            echo 'CHECK : ACE Admin SSL is enabled/disabled as expected.'
                            echo 'CHECK : HTTPS configuration matches Integration Server settings.'
                            echo 'CHECK : Certificate validity and trust.'
                            echo 'SUGGESTED FIX : Verify admin SSL settings and certificate configuration.'

                        /*
                         * 3. DEPLOYMENT / ACE PROCESSING
                         */
                        } else if (deployLog.contains('BIP8081E') ||
                                   deployLog.toLowerCase().contains('deploy') ||
                                   deployLog.toLowerCase().contains('application')) {

                            echo 'ERROR CATEGORY : ACE Application Deployment / Command Processing Error'
                            echo 'POSSIBLE REASON : ACE rejected or failed to process the BAR deployment.'
                            echo 'CHECK : BAR file contents.'
                            echo 'CHECK : Application dependencies and configuration.'
                            echo 'CHECK : Earlier BIP error messages for the actual root cause.'
                            echo 'SUGGESTED FIX : Resolve the first specific BIP error before BIP8081E.'

                        /*
                         * 4. UNKNOWN
                         */
                        } else {

                            echo 'ERROR CATEGORY : Unknown ACE Deployment Error'
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
 * Extract and print all BIP messages from ACE logs
 */
def printBipMessages(String logText) {

    echo 'Detected BIP Messages:'

    def bipMessages = []

    logText.eachLine { line ->
        if (line ==~ /.*BIP[0-9]+[A-Z]:.*/) {
            bipMessages.add(line.trim())
        }
    }

    if (bipMessages.size() > 0) {
        bipMessages.each { message ->
            echo message
        }
    } else {
        echo 'No BIP error code detected.'
    }
}
