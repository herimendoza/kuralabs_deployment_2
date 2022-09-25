pipeline {
  agent any
   stages {
    stage ('Build') {
      steps {
        sh '''#!/bin/bash
        python3 -m venv test3
        source test3/bin/activate
        pip install pip --upgrade
        pip install -r requirements.txt
        export FLASK_APP=application
        flask run &
        '''
     }
   }
    stage ('Test') {
      steps {
        sh 'echo "HELLO TEST THIS IS A WEBHOOK TEST"'
        sh '''#!/bin/bash
        source test3/bin/activate
        py.test --verbose --junit-xml test-reports/results.xml
        ''' 
      }
    
      post{
        always {
          junit 'test-reports/results.xml'
        }
       
      }
    }
     stage ('Deploy') {
       steps {
         sh '/var/lib/jenkins/.local/bin/eb deploy url-shortener-main-dev'
       }
     }
   
  }
  post{
    always{
      emailext to: "heri.mendoza9@gmail.com",
      subject: "jenkins build:${currentBuild.currentResult}: ${env.JOB_NAME}",
      body: "${currentBuild.currentResult}: Job ${env.JOB_NAME}\nMore Info can be found here: ${env.BUILD_URL}",
      attachLog: true
    }
  }
 }
