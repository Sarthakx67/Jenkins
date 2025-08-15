pipeline{
    agent any

    stages{
        stage('Create-Directory..') {
            steps{
                sh 'mkdir -p /home/ec2-user/hello'
            }
        }
        stage('Create-file..') {
            steps{
                sh "echo 'hello-world!!!' > /home/ec2-user/hello/hello.txt"
                error 'this is failed'
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
            echo 'this block will always run when job unsuccessful'
        }
    }
}