pipeline {
  agent any
  stages {
    stage('Archive') {
        when {
            branch 'main'
        }
        steps {
            withCredentials([
                string(credentialsId: 'NextTestVar', variable: 'TestVar'),
                string(credentialsId: 'NextTestVar2', variable: 'TestVar2')
            ]){
                sh 'docker build \
                    --build-arg NEXT_PUBLIC_TEST_VAR=$TestVar \
                    --build-arg NEXT_PUBLIC_TEST_VAR2=$TestVar2 \
                    -t react1 .'
                sh 'docker save react1 | gzip > react1.tar.gz'
                stash includes: 'react1.tar.gz', name: 'react1.tar.gz'
            }
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
        agent any

        when {
            branch 'main'
        }

        steps {
            withCredentials([sshUserPrivateKey(credentialsId: 'StrapiDev-Key', keyFileVariable: 'KEYFILE')]) {
            sh '""ssh -tt -i $KEYFILE ubuntu@3.73.44.106 \
                "rm -rf react1 && \
                mkdir react1 && \
                cd react1 && \
                aws s3 sync s3://jenkins-pipeline-artifacts-gdm/react1-artifacts . && docker rm -f react1 || true && docker image rm -f react1 || true  && docker image load -i react1.tar.gz && docker run -d --name react1 -p 3000:3000 react1" ""'
        }
      }
    }
  }
}