# **Deployment 2**

##### **Author: Heriberto Mendoza**

##### **Date: 24 August 2022**

---

### **Objective:**

The goal of this deployment is to become familiar with the CI/CD pipeline. Using Jenkins and Elastic Beanstalk, an app will be built, tested and deployed. This file will document the entire process.

___

#### ***1. Setting up the Jenkins EC2***

#### 1a. Jenkins instance creation:

An EC2 instance was created on AWS with certain specification. It was created with an Ubuntu image, and the firewall/securtity group was programmed to only open ports 8080, 80 and 22. A bash script was written before launching to automatically install Jenkins and the Java Runtime Environment. The following code was used:
```console
#!/bin/bash
sudo apt update
sudo apt install default-jre -y

wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo gpg --dearmor -o /usr/share/keyrings/jenkins.gpg

sudo sh -c 'echo deb [signed-by=/usr/share/keyrings/jenkins.gpg] http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'

sudo apt update
sudo apt install jenkins -y

sudo systemctl start jenkins
```

A relatively small issue was noted: a " \\" (space and backslash) had to be inserted at the end of every line before starting a new line in the user info box during the instance set up step, otherwise the script would not run.

Once Jenkins was active, the following commands were used to install python3-pip, python3.10-venv, and unzip. This was necessary to create the virtual environment and unzip packages. NOTE: I first did sudo apt update and upgrade to ensure that all the dependencies were up to date.

```console
$sudo apt install python3-pip -y
$sudo apt install python3.10-venv -y
$sudo apt install unzip -y
```

#### 1b. Jenkins user activaton:

After the necessary packages were installed, the Jenkins user account was activated in the EC2 with the following commands:
```console
# change password
$sudo passwd jenkins

# switch user with specific shell
$sudo su - jenkins -s /bin/bash
```

The IAM console on AWS was opened on a web browser to create a Jenkins user. This user, named 'EB-user', was granted programmatic access, with an attached administrator access policy. The Access Keys and Secret Key were saved for later use.

On the default Ubuntu user in the EC2, the following commands were run to install and configure AWS CLI
```console
$curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
$unzip awscliv2.zip
$sudo ./aws/install

# switch to jenkins user in EC2
$sudo su - jenkins -s /bin/bash
$aws configure
```
The AWS CLI was used to configure Jenkins and allow access to AWS. The Access and Secret Keys created in the IAM console were used here to grant Jenkins EC2 user the 'EB-user' access set up in the previous step. The region was set to us-east-1 and the output format was set to JSON.

#### 1c. EB CLI installation:

The EB CLI was installed with the following commands. It was noted that after the initial EB CLI install, the <eb> command was not able to be used within the jenkins user (`$eb --version` would fail). To solve this, the path to eb (</var/lib/jenkins/.local/bin/>) was manually added to PATH in </etc/environment/>. After this, EB CLI was able to be used in the EC2 jenkins user.
```console
$pip install awsebcli --upgrade --user
$exit

# add the path to /eb/ to the PATH
$sudo nano /etc/environment
```

---

#### ***2. Configuring Jenkins***

#### 2a. Connecting GitHub to Jenkins:

The target repositiory was forked from the Kura github to my personal github. Once forked, a web browser was used to navigate to <'ec2_ipv4':8080> to access the Jenkins web manager. After creating a login account, an access token on github was created to provide access to the repository from Jenkins. The repository was added to a multibranch pipeline and the build proceeded automatically. In addition to this, a webhook was created on GitHub to automatically push commits to Jenkins (this means that any new commits made to the repository are automatically pushed to Jenkins and Jenkins automatically builds the project).

---

#### ***3. The Jenkinsfile***

#### 3a. The 'Build' Stage:

```console
sh '''#!/bin/bash
python3 -m venv test3
source test3/bin/activate
pip install pip --upgrade
pip install -r requirements.txt
export FLASK_APP=application
flask run &
'''
```

The commands above are ran in the shell. The virtual environment <test3> is created and then activated. Before proceeding, the python package manager pip is upgraded and then used to install all the dependencies (in <requirements.txt>) necessary for the application to run. Finally, the app is ran in Flask in the background.


#### 3b. The 'Test' Stage:

```console
sh 'echo "HELLO TEST THIS IS A WEBHOOK TEST"'
sh '''#!/bin/bash
source test3/bin/activate
py.test --verbose --junit-xml test-reports/resutls.xml
'''
```

In the 'Test' stage, the first line was a line inputted by me to test a webhook that was created in the previous step. In the logs, the statement was echoed in the console output to ensure that that part of the pipeline was executed. The following shell command activates the virtual environment and the command after that `$py.test --verbose --junit-xml test-reports/results.xml` runs anything in the repository that begins with "test" and stores the results in an XML file. In this case, the test is:

```python
def test_home_page():
    response = app.test_client().get('/')
    assert response.status_code == 200
```

The above test checks if the response code for the GET request of the app root page is 200. If the request for the root page is completed successfully (200), the test will pass. In other words, an instance of the app in ran, and if the home page is returned, then the test passes.

---

#### ***4. Deploying to Elastic Beanstalk***

After the stages on Jenkins were completed, the EB CLI was used to deploy to Elastic Beanstalk (instead of manually uploading the app.zip file to the EB web interface). The following commands were used:

```console
$sudo su - jenkins -s /bin/bash
$cd /var/lib/jenkins/workspace/url_shortener_main
$eb init
...
$eb create
```

During the `$eb init` step, the server and python version were selected. The environment was created in the subsequent step, with the user selecting the environment name, environment type, etc. <insert here a screencap of the final output> After a few minutes, the app was available at a link. A glance at the AWS web interface showed that an ec2 instance had been spun up to host the flask app. Since the CLI was used, the app that was to be deployed was taken from the jenkins workspace directory in the Jenkins EC2. This eliminated the need to manually upload the app.zip file to Elastic Beanstalk.

Once the app was successfully deployed, a 'Deploy' stage was added to the Jenkinsfile. The Jenkinsfile was edited on Github, but since Github was connected to Jenkins and Jenkins had a local workspace on the EC2, the Jenkinsfile could also have been edited with 'nano'. Below is the deploy stage that was added: 

```console
sh '/var/lib/jenkins/.local/bin/eb deploy url-shortener-main-dev'
```

When the build is executed, this command would run 'eb' (a script) and deploy the environment that was created. It was noted that for the 'Deploy' stage to work, the user would have had to have done `$eb init` and `$eb create` before re-building on Jenkins. eb cannot deploy an environment that has not been created yet. Once the Elastic Beanstalk environment and app were terminated/deleted, the build would fail at the deploy stage. It would not build successfully in Jenkins until the user had used the init and create commands. However, assuming the environment was created, any changes to the source code could now be automatically deployed. Sample output from terminal after successful deployment: [terminal output](https://github.com/herimendoza/kuralabs_deployment_2/blob/1770c3ca3e3ed1a5402de86f211e353cb2587945/documentation/deploy2_success.png)

---

#### ***5. Modifying the Pipeline -- Adding an email notification***

To modify the pipeline, I decided to add an email notification step. The idea was that after every build, the build log would be sent via email. To do this, the Email Extension Plugin was needed (more versatile than the standard jenkins email notifier). In the Settings page on Jenkins, the outgoing email server and port was inputted (smtp.gmail.com and 465) along with the gmail credentials. Initially the build was successful but I was not receiving emails. Turning on the debug mode and looking at the logs showed that the email was never received because the credentials were wrong. Further investigation showed that services that use 2-factor authentication (Gmail) would only allow third party acces through an app key (similar to a Github personal access token). After setting one up, the email was successfully sent with every build. Below is the stage that was added to the Jenkinsfile

```
post {
    always {
        emailext to: "heri.mendoza9@gmail.com",
        subject: "jenkins build:${currentBuild.currentResult}: ${env.JOB_NAME}",
        body: "${currentBuild.currentResult}: Job ${env.JOB_NAME}\nMore info can be found here: ${env.BUILD_URL}",
        attachLog: true
    }

}
```

The 'always' means that that section of code will always execute, regardless of build success of fail. One can tailor notifications to be sent in case of build failure or build status change (success to failure or failure to success). The various variables used call the build status, project name, etc. The 'attachLog: true' line appends the build log to the email. Here is a sample of the log that was sent as part of the notification step: [Sample Build Log](https://github.com/herimendoza/kuralabs_deployment_2/blob/1770c3ca3e3ed1a5402de86f211e353cb2587945/documentation/build.log)

---

#### ***6. Issues***

Aside from the various issues already discussed in the individual sections, one general issue that comes to mind is syntax. Over the course of this deployment, there were times when the installation script didn't run or Jenkins wasn't added to the sources list properly or the build failed at various stages. Oftentimes, the problem was syntax: missing a forward slash here, a comma there. In the Jenkinsfile in particular, a misplaced bracket, quote mark or comma could cause the pipeline to fail. Care should be taken to ensure that proper syntax is observed.
