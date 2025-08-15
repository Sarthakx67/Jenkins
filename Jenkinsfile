pipeline{
    agent any

    stages{
        stage('Create-Directory..') {
            steps{
                sh 'mkdir -p /home/ec2-user/hello.txt'
            }
        }
        stage('print-hello-world..') {
            steps{
                echo 'hello-world!!!'
            }
        }
        stage('bye..') {
            steps{
                echo 'byeeeee.....'
            }
        }
    }
}