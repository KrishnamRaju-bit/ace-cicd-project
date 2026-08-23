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
    }
}
