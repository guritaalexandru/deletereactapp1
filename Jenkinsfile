pipeline {
  agent {
    docker {
      image 'docker:19.03.12'
    }
  }
  stages {
    stage('Archive') {
        when {
            branch 'main'
        }
        steps {
            sh 'docker build -t react1 .'
            sh 'docker save react1 | gzip > react1.tar.gz'
            stash includes: 'react1.tar.gz', name: 'react1.tar.gz'
        }
    }

    stage('Upload to S3') {
        options {
            withAWS(credentials: 'AWS_CREDENTIALS', region: 'eu-central-1')
        }

        when {
            branch 'main'
        }

        agent {
            docker { image 'amazon/aws-cli:latest' }
        }

        steps {
            unstash 'react1.tar.gz'
            s3Delete(bucket: 'jenkins-pipeline-artifacts-gdm', path: 'react1-artifacts/')
            s3Upload(file: 'react1.tar.gz', bucket: 'jenkins-pipeline-artifacts-gdm', path: 'react1-artifacts/')
        }
    }

    stage('Deploy') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'StrapiDev-Key', keyFileVariable: 'KEYFILE')]) {
          sh 'aws ssm get-parameters --names /app/env_vars --with-decryption --region us-west-2 | jq -r ".Parameters[] | .Value" > env_vars'
          sh 'scp -i my-key.pem env_vars ubuntu@my-instance:/tmp'
          sh 'ssh -i my-key.pem ubuntu@my-instance "docker load -i /tmp/myapp_$BUILD_NUMBER.tar.gz"'
          sh 'ssh -i my-key.pem ubuntu@my-instance "docker stop myapp || true"'
          sh 'ssh -i my-key.pem ubuntu@my-instance "docker rm myapp || true"'
          sh 'ssh -i my-key.pem ubuntu@my-instance "docker run -d --name myapp -e env_file=./env_vars -p 80:80 myapp:$BUILD_NUMBER"'
        }
      }
    }
  }
}