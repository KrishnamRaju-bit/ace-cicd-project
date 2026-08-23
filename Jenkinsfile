pipeline {
    agent any

    stages {

        stage('Test Pipeline') {
            steps {
                echo 'ACE CI/CD Pipeline Started Successfully'
            }
        }

        stage('Check ACE Environment') {
            steps {
                bat '''
                call "C:\\Program Files\\IBM\\ACE\\13.0.8.0\\server\\bin\\mqsiprofile.cmd"
                echo ACE Environment Loaded
                where ibmint
                ibmint --version
                '''
            }
        }

        stage('Build BAR') {
            steps {
                bat '''
                call "C:\\Program Files\\IBM\\ACE\\13.0.8.0\\server\\bin\\mqsiprofile.cmd"

                if not exist build mkdir build

                ibmint package ^
                  --input-path "%WORKSPACE%" ^
                  --output-bar-file "%WORKSPACE%\\build\\ACE_Jenkins.bar" ^
                  --project ACE_Jenkins

                echo BAR Build Completed
                dir "%WORKSPACE%\\build"
                '''
            }
        }

        stage('Deploy BAR and Analyze Errors') {
            steps {
                script {

                    def deployStatus = bat(
                        returnStatus: true,
                        script: '''
                        call "C:\\Program Files\\IBM\\ACE\\13.0.8.0\\server\\bin\\mqsiprofile.cmd"

                        ibmint deploy ^
                          --input-bar-file "%WORKSPACE%\\build\\ACE_Jenkins.bar" ^
                          --output-host localhost ^
                          --output-port 7600 ^
                          --https ^
                          --insecure ^
                          > "%WORKSPACE%\\build\\deploy.log" 2>&1

                        exit /b %ERRORLEVEL%
                        '''
                    )

                    bat '''
                    echo.
                    echo ================================
                    echo ACE DEPLOYMENT LOG
                    echo ================================
                    type "%WORKSPACE%\\build\\deploy.log"
                    '''

                    if (deployStatus != 0) {

                        echo '================================'
                        echo 'ACE DEPLOYMENT FAILED'
                        echo 'Starting Automatic Error Analysis'
                        echo '================================'

                        bat '''
                        echo.
                        echo Detected BIP Messages:
                        findstr /R "BIP[0-9][0-9]*[A-Z]:" "%WORKSPACE%\\build\\deploy.log"
                        exit /b 0
                        '''

                        def connectivityError = bat(
                            returnStatus: true,
                            script: '''
                            findstr /C:"BIP1921S" /C:"BIP8032E" "%WORKSPACE%\\build\\deploy.log" >nul
                            '''
                        )

                        if (connectivityError == 0) {

                            echo 'ERROR CATEGORY : ACE Connectivity / Admin REST Error'
                            echo 'POSSIBLE REASON : Integration Server cannot be reached.'
                            echo 'CHECK : Integration Server is running.'
                            echo 'CHECK : Host and Admin REST port are correct.'
                            echo 'CHECK : HTTP/HTTPS protocol matches ACE Admin SSL configuration.'
                            echo 'SUGGESTED FIX : Verify server status, port 7600 and Admin SSL settings.'

                        } else {

                            def processingError = bat(
                                returnStatus: true,
                                script: '''
                                findstr /C:"BIP8081E" "%WORKSPACE%\\build\\deploy.log" >nul
                                '''
                            )

                            if (processingError == 0) {
                                echo 'ERROR CATEGORY : ACE Command Processing Error'
                                echo 'POSSIBLE REASON : ibmint deploy command failed.'
                                echo 'SUGGESTED FIX : Check the BIP messages printed before BIP8081E for the root cause.'
                            } else {
                                echo 'ERROR CATEGORY : Unknown ACE Deployment Error'
                                echo 'SUGGESTED FIX : Review detected BIP messages in deploy.log.'
                            }
                        }

                        error('ACE BAR deployment failed. Automatic error analysis completed.')

                    } else {

                        echo '================================'
                        echo 'ACE BAR DEPLOYMENT SUCCESSFUL'
                        echo '================================'
                    }
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'build/deploy.log',
                             allowEmptyArchive: true
        }
    }
}
