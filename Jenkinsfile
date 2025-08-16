pipeline{
    agent any

    stages{
        stage('mkdir..') {
            steps{
                sh 'mkdir -p /home/ec2-user/hello'
            }
        }
        stage('running multiple commands..') {
            steps{
                sh '''
                ls -ltr
                pwd
                echo "hello scripts"
                '''
            }
        }
        stage('bye..') {
            steps{
                echo 'byeeeee.....'
            }
        }
    }
    post{
        always {
            echo 'this block will always run wheather job is success or not'
        }
        success {
            echo 'this block will run when job is success'
        }
        failure {
            echo 'this block will always run when job is unsuccessful'
        }
    }
}