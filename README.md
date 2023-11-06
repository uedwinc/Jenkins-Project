# Jenkins Project

This project will incorporate:
- Using Maven to build a pom.xml into a '.war' app file
- Integrating Sonarqube for code analysis and setting up quality gate
- Pushing the artifact into Nexus repository
- Integrating Slack notification
- Building a docker image for our application
- Pushing the image to ECR
- Deploying the image with ECS
- Configuring a Jenkins-Github webhook to build at push

## Installations

### Starting and Configuring the Jenkins Server

1. Launch a Jenkins Server EC2 instance on AWS (I used Ubuntu AMI on t2.small).

2. During launch, provision [user data]()

- This includes:
    - A bash installation of jenkins as seen in the documentation (https://www.jenkins.io/doc/book/installing/linux/#debianubuntu)
    - Maven installation
    - JDK11 installation

3. Now, ssh into the jenkins server (username=ubuntu)

4. We may also need java8 so install

- First, do `apt-get update`

- Then `apt install openjdk-8-jdk -y`

5. Jenkins works on port 8080 so make sure that is opened in the security group on AWS

6. Confirm jenkins installation using `systemctl status jenkins`. The output of this also gives you password to jenkins.

![jenkins running]()

7. Now, open jenkins console on the browser with ip address and port and enter the password.

8. Customize jenkins with suggested plugins and user information. Then launch the jenkins console.

![jenkins console]()

9. Go to 'Tools' under 'Manage Jenkins' and setup JDK, git and maven.

    - Path for java installation is /usr/lib/jvm/

![jdk]()

![git]()

![maven]()

10. Installing Plugins

- Install the following plugins:

    1. Nexus Artifact Uploader
    2. SonarQube Scanner
    3. Pipeline utility steps
    4. Pipeline maven integration
    5. Timestamp plugins: 'Build Timestamp' and 'Zentimestamp'

### Starting and Configuring Nexus Server

1. Launch a Nexus Server instance on AWS (I used CentOS 7 from AWS Marketplace)
    - Although Nexus requires 4vcpu and 4gb ram, I will stick to t3.meduim to minimize cost.

2. Provision [user data]() to install nexus, install JDK, set up environment variables and start nexus service

3. SSH into the Nexus Server (username=centos)

4. Confirm that Nexus is running `systemctl status nexus`

![nexus running]()

5. Nexus runs on port 8081, so open this in the security group on AWS

6. Now, access Nexus console on the browser with the IP address and port

7. Sign-in with username "admin" and password as specified in the path on the sign-in screen

![nexus sign in]()

![nexus console]()

### Starting and Configuring Sonarqube Server

Sonarqube helps in code quality analysis

1. Launch a Sonarqube Server instance on AWS (I used Ubuntu AMI and t3.medium)

2. Provision [user data]()

3. SSH into the server and confirm installation using `systemctl status sonarqube`

![sonarqube running]()

4. Sonarqube runs on port 80, so open that in the security group

5. Access Sonarqube console on the browser with the IP address (No need to specify port as 80 is the default HTTP port)

6. Log in (username=admin, password=admin)

![sonar sign in]()

---

## Simple Maven Build

1. Define a 'new item' on jenkins and select 'pipeline'

2. Configure a new pipeline by choosing 'Pipeline script' option

3. Create a Jenkinsfile

- For archiving artifact, integrate an archiveArtifacts process
    - See Documentation at https://www.jenkins.io/doc/pipeline/steps/core/#archiveartifacts-archive-the-artifacts

4. Paste the Jenkinsfile code on the console

5. Save and Build

![mvn build]()

6. On the jenkins server, switch to jenkins user (created by default when jenkins was installed) `su jenkins`
	- Go to `/var/lib/jenkins/workspace/` to see all archived builds.
	- Go to `/targets` of any build to see the artifact that was generated.

        ![terminal artifact]()

	- This can also be found on the jenkins console workspace.

        ![console artifact]()

---

## Configure Sonarqube Analysis

1. Under "Manage Jenkins", go to "Tools"

    - Go to "SonarQube Scanner" section and add sonarqube scanner
	    - Name eg sonar4.7
	    - Select the appropriate version. //Latest version may not work so it is better to stick to more stable version//
	    - Save.

![sonar4.7]()

2. Again, go to "System" under "Manage Jenkins" and configure
	
    - Under "SonarQube Servers", check the box for 'Environment variables'.
	
    - Click on 'Add sonarqube'.

		- name eg sonarqube

		- server url is http://private-ip-address-of-the-sonarqube-server

		- server authentication token:
			
            + In the sonarqube web console, generate a token
				- Go to profile name > my account > security
				- Enter any name eg 'jenkins' and generate token
				- Copy the generated token

            ![token]()
			
            + On the jenkins console, click on "add", select jenkins.
			
            + This opens a jenkins credentials window
				- Kind: select "secret text" from the dropdown
				- Secret: Paste the generated token
				- ID: eg JenkinsSonar
				- Description: eg JenkinsSonarIntegration
				- Finally "Add".
			+ Now, for the server authentication token, select the "JenkinsSonarIntegration"
	- Save.

![sonar servers]()

3. Integrate checkstyle analysis in the Jenkinsfile 
	- Documentation: https://maven.apache.org/plugins/maven-checkstyle-plugin/usage.html

4. Then, use sonar analysis to view the report of the checkstyle and other plugin analysis

5. Defining the -DSonar variables
	+ Log into jenkins server as the jenkins user `su jenkins`
		- Do `ls` to see the workspace directory
		- `cd` into the workspace directory and do `ls`
		- `cd` into the PAAC_Checkstyle_Analysis 
		- Next, `cd` into targets and do `ls`
		- Here, you will find all the DSonar defined variables
	+ So, it is necessary to execute a simple maven build inorder to see these paths in the server

6. Run a new pipeline

![sonar p]()

7. On the sonarqube web console, you can see the results of the code analysis.

![sonar c]()

## Setting up Quality Gate

This is used to set condition for pipeline success depending on code quality

1. On the sonarqube web console, on the top menu, click "Quality Gates"
	- Click create and give it a name eg JenkinsSonarGate. Then save.
	- Add conditions:
		- On overall code
		- Quality gate fails when: select eg Bugs
		- Specify operator and value, eg 50 or lower

![gate50]()

2. On the sonarqube web console, on the top menu, click "Projects"
	- Click on the project, that is the MavenBuildScan
	- On the top right, click on "Project Settings" dropdown and select "Quality Gate"
	- Choose the quality gate that was defined

![gate-add]()

	- Go to project settings dropdown again and select "Webhooks"
	- Click "Create"
		+ Name: eg JenkinsWebhook
		+ URL: http://private-ip-of-the-jenkins-server:8080/sonarqube-webhook
		+ No secret so Create

![webh]()

2. On the Jenkinsfile, add stage for quality gate

3. Run the pipeline (it would fail if quality gate conditions are not met)

![qg fail]()

![qg fail console]()

4. Adjust the quality gate above 82 to see if it will succeed

![qg adjust]()

- Pipeline Succeeded

![qg succeed]()

## Integrating Nexus Artifact Push

1. Configure version
	- On the jenkins server, go to manage jenkins > system
	- Under Global properties, set-up BUILD_TIMESTAMP
	- Save

![btstamp]()

2. Create repository on Nexus console
	- Go to repositories and create a maven2(hosted) repository
		- Give it a name and save
	- Go to roles and create a new role (nexus role)
		- Under priviledges, search repository name and select view* role
		- Then save
	- Go to user and create a new user
		- Attach the role created to it

        ![nx user]()

3. Configure credentials id
- This is a way to pass in nexus credentials into the pipeline.
- It is achieved by configuring credentials on jenkins console with nexus-id
	- Go to manage jenkins > credentials
	- Under credentials, click on system.
	- Click on Global credentials (unrestricted) and then add credentials
		+ Specify username with password
		+ Username = id of the user created on nexus
		+ Enter password
		+ Specify an ID and description (note the ID specified)
		+ Create

![j m cred]()

4. Adjust quality gate so the pipeline doesn't fail

5. Create new project on jenkins, paste pipeline code and build.

![nex pipe]()

- See artifact on Nexus

![nx art console]()

## Configure Slack notification

1. On jenkins, install "Slack Notification" plugin

2. Go to any slack channel app section (eg https://devopsacad.slack.com/apps) (Be sure to select the appropriate workspace)
	- Search 'jenkins' in the search bar
	- Choose 'Jenkins CI'

![slack dev]()

	- Click on 'Add to slack'
	- Choose a channel in the workspace to post notifications to
	- Click on 'Add jenkins CI integration'
	- Scroll down and copy the token
	- Save

3. Go to jenkins console > manage jenkins > system
	- Scroll down to "Slack" section
	- Specify workspace name and default channel
	- Under credentials, click "Add"
		+ Change kind to "secret text" and paste the token
		+ Enter any appropriate ID and description (copy the ID)
		+ Add

        ![jk creds]()

	- Choose the newly created credential under credentials
	- Save

    ![slack]

4. Edit the jenkinsfile and add a post-build action for slack notification

5. Create a new item project on jenkins and run the pipeline on it.

![slack success]()

![slack actual]()

## Build Docker Image and Push to ECR

1. On the Jenkins Server, install awscli
	- Install unzip first if not installed
	- Install awscli following installation guide here: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html
	- Do `aws --version` to confirm

2. Install docker on the Jenkins Server
	- Follow the installation guide here: https://docs.docker.com/engine/install/ubuntu/
	- Do `docker --version` to confirm

3. Give the jenkins user permissions to execute docker commands by adding it to the docker group
	- `usermod -a -G docker jenkins`
	- `id jenkins` to confirm

4. Create an IAM user and role for jenkins

- Go to the IAM page
	- Click users > Add users
	- Give it a name eg jenkins and click next (don't check console management access)

    ![aws jk user]()

	- Attach policies directly and attach the following policies
		+ AmazonEC2ContainerRegistryFullAccess
		+ AmazonECS_FullAccess
	- Click next and create user

	- Click on the jenkins user just created
		+ Go to Security credentials
		+ Create access key. Check Command line interface (CLi). Then create.

5. Configure jenkins

- Go to manage jenkins > plugins
	+ Install the following plugins
		- Pipeline: AWS Steps
		- Docker Pipeline
		- Amazon ECR
		- Amazon Web Services SDK::All
		- CloudBees Docker Build and Publish
- Go to manage jenkins > credentials
	+ Click on system > Global credentials > Add credentials
		- Under kind, select AWS credentials //AWS Credentials for ECR & ECS//
		- Enter ID and description
		- Paste access and secret key from the IAM user role created
		- Create

6. Configure Amazon Elastic Container Registry

- Open the ECR page
	- Go to 'Repositories' and Create repository
		+ Check 'private'
		+ Give it a name and then create repository

7. Update the Jenkinsfile

9. Create a new pipeline project on jenkins, paste the jenkinsfile, save and build. //May need to restart jenkins service and docker engine before build//