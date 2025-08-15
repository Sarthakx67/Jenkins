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
                sh 'touch /home/ec2-user/hello/hello.txt'
            }
        }
        stage('print-hello-world..') {
            steps{
                sh 'echo "hello-world!!! > /home/ec2-user/hello/hello.txt"'
            }
        }
        stage('bye..') {
            steps{
                echo 'byeeeee.....'
            }
        }
    }
}