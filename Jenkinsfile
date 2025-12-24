pipeline {
    agent { node { label 'AGENT-1' } } // to give agent label 
    options {
        timeout(time: 1, unit: 'HOURS')  // how long our pipeline should run for 
    }
    // triggers { //// how frequently our pipeline/build should run
    //     cron('* * * * *') //// trigger the build to run every minute.
    // }
    environment { 
        USER = 'Sarthak' // declaring environment variables that is accessible through all stages in a pipeline
    }
    parameters { // input parameters for pipeline that we can specify when we manually trigger a pipeline
        string(name: 'PERSON', defaultValue: 'Mr Jenkins', description: 'Who should I say hello to?') // string(...): A simple text field.
        text(name: 'BIOGRAPHY', defaultValue: '', description: 'Enter some information about the person') // text(...): A multi-line text area.
        booleanParam(name: 'TOGGLE', defaultValue: true, description: 'Toggle this value') // booleanParam(...): A checkbox (true/false) option.
        choice(name: 'CHOICE', choices: ['One', 'Two', 'Three'], description: 'Pick something') // choice(...): A dropdown menu with predefined options.
        password(name: 'PASSWORD', defaultValue: 'SECRET', description: 'Enter a password') // password(...): A hidden text field for sensitive information.
    }
    stages {
        stage('Build') {
            steps {
                echo 'Building..'
              sh '''
                ls -ltr
                pwd
                echo "Hello from GitHub Push webhook event"
                printenv
              '''
            }
        }
        stage('Test') {
            steps {
                echo 'Testing..'
            }
        }
        stage('Deploy') {
            steps {
                echo 'Deploying....'
                //error 'this is failed'
            }
        }

        stage('Example') {
            environment { 
                AUTH = credentials('ssh-auth') 
            }
            steps {
                sh 'printenv'
            }
        } 
        stage('Params') { // using parameters
            steps { 
                echo "Hello ${params.PERSON}"

                echo "Biography: ${params.BIOGRAPHY}"

                echo "Toggle: ${params.TOGGLE}"

                echo "Choice: ${params.CHOICE}"

                echo "Password: ${params.PASSWORD}"
            }
        }
        stage('Input') { // this stage pauses the pipeline and wait for user to approve for continuation
            input {
                message "Should we continue?"
                ok "Yes, we should."
                submitter "alice,bob"
                parameters {
                    string(name: 'PERSON', defaultValue: 'Mr Jenkins', description: 'Who should I say hello to?')
                }
            }
            steps {
                echo "Hello, ${PERSON}, nice to meet you."
            }
        }
        stage('PROD Deploy'){
            when {
                environment name: 'USER', value: 'Sarthak' //This stage will only run if the environment variable USER has the value Sarthak
            }
            steps{
                echo "deploying to PROD"
            }
        }
        stage('Parallel Stage') {
 
            parallel { // parallel will insure stages to run simutaniously
                stage('Branch A') {
                    steps {
                        echo "On Branch A"
                        sh 'sleep 10'
                    }
                }
                stage('Branch B') {
                    steps {
                        echo "On Branch B"
                        sh 'sleep 10'
                    }
                }
                stage('Branch C') { // this will run parallel to (branch A and B) but will run stages in Branch C sequentially  
                    stages {
                        stage('Nested 1') {
                            steps {
                                echo "In stage Nested 1 within Branch C"
                                sh 'sleep 10'
                            }
                        }
                        stage('Nested 2') {
                            steps {
                                echo "In stage Nested 2 within Branch C"
                                sh 'sleep 10'
                            }
                        }
                    }
                }
            }
        }
    }
    post { 
        always { 
            echo 'I will always run whether job is success or not'
        }
        success{
            echo 'I will run only when job is success'
        }
        failure{
            echo 'I will run when failure'
        }
    }
}