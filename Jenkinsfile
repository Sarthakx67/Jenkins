pipeline{
    agent any

    stages{
        stage('Create-Directory..') {
            steps{
                sh '''
                sudo su -
                mkdir -p /home/ec2-user/hello
                '''
            }
        }
        stage('Create-file..') {
            steps{
                sh "echo 'hello-world!!!' > /home/ec2-user/hello/hello.txt"
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