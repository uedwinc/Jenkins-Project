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

---

## Tools and Technologies

![docker](https://img.shields.io/badge/docker-2496ED?style=for-the-badge&labelColor=black&logo=docker&logoColor=2496ED) ![jenkins](https://img.shields.io/badge/jenkins-D24939?style=for-the-badge&labelColor=black&logo=jenkins&logoColor=D24939) ![amazon ec2](https://img.shields.io/badge/amazonec2-FF9900?style=for-the-badge&labelColor=black&logo=amazonec2&logoColor=FF9900) ![Maven](https://img.shields.io/badge/apachemaven-3-C71A36?style=for-the-badge&logo=apachemaven) ![Sonatype](https://img.shields.io/badge/sonatype-3-1B1C30?style=for-the-badge&logo=sonatype)

---

## Installations

### Starting and Configuring the Jenkins Server

1. Launch a Jenkins Server EC2 instance on AWS (I used Ubuntu AMI on t2.small).

2. During launch, provision [user data](https://github.com/uedwinc/Jenkins-Project/blob/main/userdata/jenkins-setup.sh)

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

![jenkins running](https://github.com/uedwinc/Jenkins-Project/blob/main/images/jenkins%20running.png)

7. Now, open jenkins console on the browser with ip address and port and enter the password.

8. Customize jenkins with suggested plugins and user information. Then launch the jenkins console.

![jenkins console](https://github.com/uedwinc/Jenkins-Project/blob/main/images/jenkins%20console.png)

9. Go to 'Tools' under 'Manage Jenkins' and setup JDK, git and maven.

- Path for java installation is /usr/lib/jvm/

![jdk](https://github.com/uedwinc/Jenkins-Project/blob/main/images/jdk.png)

![git](https://github.com/uedwinc/Jenkins-Project/blob/main/images/git.png)

![maven](https://github.com/uedwinc/Jenkins-Project/blob/main/images/maven.png)

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

2. Provision [user data](https://github.com/uedwinc/Jenkins-Project/blob/main/userdata/nexus-setup.sh) to install nexus, install JDK, set up environment variables and start nexus service

3. SSH into the Nexus Server (username=centos)

4. Confirm that Nexus is running `systemctl status nexus`

![nexus running](https://github.com/uedwinc/Jenkins-Project/blob/main/images/nexus%20running.png)

5. Nexus runs on port 8081, so open this in the security group on AWS

6. Now, access Nexus console on the browser with the IP address and port

7. Sign-in with username "admin" and password as specified in the path on the sign-in screen

![nexus sign in](https://github.com/uedwinc/Jenkins-Project/blob/main/images/nexus%20sign%20in.png)

![nexus console](https://github.com/uedwinc/Jenkins-Project/blob/main/images/nexus%20console.png)

### Starting and Configuring Sonarqube Server

Sonarqube helps in code quality analysis

1. Launch a Sonarqube Server instance on AWS (I used Ubuntu AMI and t3.medium)

2. Provision [user data](https://github.com/uedwinc/Jenkins-Project/blob/main/userdata/sonar-setup.sh)

3. SSH into the server and confirm installation using `systemctl status sonarqube`

![sonarqube running](https://github.com/uedwinc/Jenkins-Project/blob/main/images/sonarqube%20running.png)

4. Sonarqube runs on port 80, so open that in the security group

5. Access Sonarqube console on the browser with the IP address (No need to specify port as 80 is the default HTTP port)

6. Log in (username=admin, password=admin)

![sonar sign in](https://github.com/uedwinc/Jenkins-Project/blob/main/images/sonar%20sign%20in.png)

---

## Simple Maven Build

1. Define a 'new item' on jenkins and select 'pipeline'

2. Configure a new pipeline by choosing 'Pipeline script' option

3. Create a Jenkinsfile

- For archiving artifact, integrate an archiveArtifacts process
    - See Documentation at https://www.jenkins.io/doc/pipeline/steps/core/#archiveartifacts-archive-the-artifacts

4. Paste the Jenkinsfile code on the console

5. Save and Build

![mvn build](https://github.com/uedwinc/Jenkins-Project/blob/main/images/mvn%20build.png)

6. On the jenkins server, switch to jenkins user (created by default when jenkins was installed) `su jenkins`
   
	- Go to `/var/lib/jenkins/workspace/` to see all archived builds.

	- Go to `/targets` of any build to see the artifact that was generated.

        ![terminal artifact](https://github.com/uedwinc/Jenkins-Project/blob/main/images/terminal%20artifact.png)

	- This can also be found on the jenkins console workspace.

        ![console artifact](https://github.com/uedwinc/Jenkins-Project/blob/main/images/console%20artifact.png)

---

## Configure Sonarqube Analysis

1. Under "Manage Jenkins", go to "Tools"

    - Go to "SonarQube Scanner" section and add sonarqube scanner
	    - Name eg sonar4.7
	    - Select the appropriate version. //Latest version may not work so it is better to stick to more stable version//
	    - Save.

![sonar4.7](https://github.com/uedwinc/Jenkins-Project/blob/main/images/sonar4.7.png)

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

            ![token](https://github.com/uedwinc/Jenkins-Project/blob/main/images/token.png)
			
            + On the jenkins console, click on "add", select jenkins.
			
            + This opens a jenkins credentials window
				- Kind: select "secret text" from the dropdown
				- Secret: Paste the generated token
				- ID: eg JenkinsSonar
				- Description: eg JenkinsSonarIntegration
				- Finally "Add".
			+ Now, for the server authentication token, select the "JenkinsSonarIntegration"
	- Save.

![sonar servers](https://github.com/uedwinc/Jenkins-Project/blob/main/images/sonar%20servers.png)

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

![sonar p](https://github.com/uedwinc/Jenkins-Project/blob/main/images/sonar%20p.png)

7. On the sonarqube web console, you can see the results of the code analysis.

![sonar c](https://github.com/uedwinc/Jenkins-Project/blob/main/images/sonar%20c.png)

## Setting up Quality Gate

This is used to set condition for pipeline success depending on code quality

1. On the sonarqube web console, on the top menu, click "Quality Gates"
	- Click create and give it a name eg JenkinsSonarGate. Then save.
	- Add conditions:
		- On overall code
		- Quality gate fails when: select eg Bugs
		- Specify operator and value, eg 50 or lower

![gate50](https://github.com/uedwinc/Jenkins-Project/blob/main/images/gate50.png)

2. On the sonarqube web console, on the top menu, click "Projects"
	- Click on the project, that is the MavenBuildScan
	- On the top right, click on "Project Settings" dropdown and select "Quality Gate"
	- Choose the quality gate that was defined

![gate-add](https://github.com/uedwinc/Jenkins-Project/blob/main/images/gate-add.png)

3. Go to project settings dropdown again and select "Webhooks"
	- Click "Create"
		+ Name: eg JenkinsWebhook
		+ URL: http://private-ip-of-the-jenkins-server:8080/sonarqube-webhook
		+ No secret so Create

![webh](https://github.com/uedwinc/Jenkins-Project/blob/main/images/webh.png)

4. On the Jenkinsfile, add stage for quality gate

5. Run the pipeline (it would fail if quality gate conditions are not met)

![qg fail](https://github.com/uedwinc/Jenkins-Project/blob/main/images/qg%20fail.png)

![qg fail console](https://github.com/uedwinc/Jenkins-Project/blob/main/images/qg%20fail%20console.png)

6. Adjust the quality gate above 82 to see if it will succeed

![qg adjust](https://github.com/uedwinc/Jenkins-Project/blob/main/images/qg%20adjust.png)

- Pipeline Succeeded

![qg succeed](https://github.com/uedwinc/Jenkins-Project/blob/main/images/qg%20succeed.png)

## Integrating Nexus Artifact Push

1. Configure version
	- On the jenkins server, go to manage jenkins > system
	- Under Global properties, set-up BUILD_TIMESTAMP
	- Save

![btstamp](https://github.com/uedwinc/Jenkins-Project/blob/main/images/btstamp.png)

2. Create repository on Nexus console
	- Go to repositories and create a maven2(hosted) repository
		- Give it a name and save
	- Go to roles and create a new role (nexus role)
		- Under priviledges, search repository name and select view* role
		- Then save
	- Go to user and create a new user
		- Attach the role created to it

        ![nx user](https://github.com/uedwinc/Jenkins-Project/blob/main/images/nx%20user.png)

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

![j m cred](https://github.com/uedwinc/Jenkins-Project/blob/main/images/j%20m%20cred.png)

4. Adjust quality gate so the pipeline doesn't fail

5. Create new project on jenkins, paste pipeline code and build.

![nex pipe](https://github.com/uedwinc/Jenkins-Project/blob/main/images/nex%20pipe.png)

- See artifact on Nexus

![nx art console](https://github.com/uedwinc/Jenkins-Project/blob/main/images/nx%20art%20console.png)

## Configure Slack notification

1. On jenkins, install "Slack Notification" plugin

2. Go to any slack channel app section (eg https://devopsacad.slack.com/apps) (Be sure to select the appropriate workspace)
	- Search 'jenkins' in the search bar
	- Choose 'Jenkins CI'

![slack dev](https://github.com/uedwinc/Jenkins-Project/blob/main/images/slack%20dev.png)

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

        ![jk creds](https://github.com/uedwinc/Jenkins-Project/blob/main/images/jk%20creds.png)

	- Choose the newly created credential under credentials
	- Save

    ![slack](https://github.com/uedwinc/Jenkins-Project/blob/main/images/slack.png)

4. Edit the jenkinsfile and add a post-build action for slack notification

5. Create a new item project on jenkins and run the pipeline on it.

![slack success](https://github.com/uedwinc/Jenkins-Project/blob/main/images/slack%20success.png)

![slack actual](https://github.com/uedwinc/Jenkins-Project/blob/main/images/slack%20actual.png)

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

    ![aws jk user](https://github.com/uedwinc/Jenkins-Project/blob/main/images/aws%20jk%20user.png)

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

![aws creds](https://github.com/uedwinc/Jenkins-Project/blob/main/images/aws%20creds.png)

6. Configure Amazon Elastic Container Registry

- Open the ECR page
	- Go to 'Repositories' and Create repository
		+ Check 'private'
		+ Give it a name and then create repository

![aws repo](https://github.com/uedwinc/Jenkins-Project/blob/main/images/aws%20repo.png)

7. Update the Jenkinsfile

9. Create a new pipeline project on jenkins, paste the jenkinsfile, save and build. # May need to restart jenkins service and docker engine before build

![ecr success](https://github.com/uedwinc/Jenkins-Project/blob/main/images/ecr%20success.png)

- Go to ECR page to see the image

![ecr image success](https://github.com/uedwinc/Jenkins-Project/blob/main/images/ecr%20image%20success.png)

- ECR scan result

![ecr scan](https://github.com/uedwinc/Jenkins-Project/blob/main/images/ecr%20scan.png)

- `docker images` on the server will also show the images

![ecr docker images](https://github.com/uedwinc/Jenkins-Project/blob/main/images/ecr%20docker%20images.png)

- Check notification on slack

![ecr slack](https://github.com/uedwinc/Jenkins-Project/blob/main/images/ecr%20slack.png)

## Deploy Image with ECS

1. Create Amazon Elastic Container Service (ECS) Cluster

- Go to ECS page (Ensure 'New ECS Experience' is toggled on)
	- Go to Clusters and Create cluster
		- Give the cluster a name
		- Under networking, you can leave the defaults (it's advised in production to use non-defaults)
		- For Infrastructure, check AWS fargate (serverless)
		- Under monitoring, toggle on 'use container insights'
		- Add tags and create.

![cluster name](https://github.com/uedwinc/Jenkins-Project/blob/main/images/cluster%20name.png)

2. Go to Task definitions and create new task definition
	- Give it a family name
	- Launch type is fargate
	- Leave vcpu and memory as default (1vcpu 2gb memory)
	- For task execution role, select create new role
	- Container - 1 container details
		+ Give it a name
		+ Image URI is the ECR uri (Amazon ECR > Repositories)
		+ Container port is 8080
	- Create (This takes some time to create)

![task definition](https://github.com/uedwinc/Jenkins-Project/blob/main/images/task%20definition.png)

3. After creation, click on the ecsTaskExecutionRole link
	- Make sure the following roles are added (Add permissions > Attach policies)
		+ CloudWatchLogsFullAccess (CloudwatchFullAccess)
		+ AmazonECSTaskExecutionRolePolicy

![task execution](https://github.com/uedwinc/Jenkins-Project/blob/main/images/task%20execution.png)

4. Under Clusters, go to Services and click create
	- Check capacity provider strategy
	- Check use custom (Advanced)
	- Leave defaults (capacity provider=fargate, base=0, weight=1, platform version=latest)
	- For deployment configuration:
		+ Application type = service
		+ Select family. Revision=1(latest)
		+ Give service name
		+ Service type = Replica
		+ Desired tasks = 2 (or just 1)
	- Deployment failure detection
		+ Uncheck 'use the amazon ECS deployment circuit breaker
	- Under networking
		+ Specify vpc and subnets
		+ Create new security group
			- Specify name and description
			- Open up port 8080 (Since we have tomcat running in the container)
			- Open port 80 (this is the port the load balancer will be using for outside connection)
	- For load balancing
		+ Type is application load balancer
		+ Create a new load balancer
		+ Give the load balancer a name
		+ Choose container to load balance
		+ For listener, create new listener and specify port 80
		+ For target group, create new target group
			- Specify a target group name
			- Health check path = /login
	- Create
	- The instances will appear under target groups but won't show up under instances since they're serverless

![cluster service](https://github.com/uedwinc/Jenkins-Project/blob/main/images/cluster%20service.png)

5. Under Clusters > cluster-name > services > service-name, go to Networking and copy the DNS name (which can also be found under load balancers) and open in browser

![devopsacad](https://github.com/uedwinc/Jenkins-Project/blob/main/images/devopsacad.png)

6. Update the Jenkinsfile to update ECR and deploy latest build to ECS

7. Create an ECS pipeline project and build.

![ecs success](https://github.com/uedwinc/Jenkins-Project/blob/main/images/ecs%20success.png)

- In the clusters page and target groups, you can see the clusters draining and updating.

![draining1](https://github.com/uedwinc/Jenkins-Project/blob/main/images/draining1.png)

![draining2](https://github.com/uedwinc/Jenkins-Project/blob/main/images/draining2.png)

8. Confirm the application is still running on the browser and slack notification.

![ecs slack](https://github.com/uedwinc/Jenkins-Project/blob/main/images/ecs%20slack.png)

## Configuring Webhook to Build at Push

1. Configure repository settings

- Under the Jenkins-Project repository, click settings
	- Click webhooks > Add webhook
		+ Payload url = http://jenkins-ip:8080/github-webhook/
		+ Content type = application/json
		+ For events to trigger, either 'just the push event' or you can 'select individual events'
		+ Check active and then add.
	- There should be a green check to confirm. If not check security group of jenkins server to ensure it's accessible publicly.

![hook success](https://github.com/uedwinc/Jenkins-Project/blob/main/images/hook%20success.png)

2. On the Jenkins console, go to 'Manage jenkins' > 'security'
	- Scroll down and find 'Git Host Key verification Configuration'
	- Under the dropdown for host key verification strategy, select 'Accept first connection'
	- Save

3. Create a new pipeline project item on jenkins
	- Under definition, select Pipeline script from SCM
	- Under SCM, select git
	- Paste the https url (ssh url if it is private) of the git repository
		
        **Follow these steps for private repository**
        //+ Under credentials, click Add (jenkins)
			- Change the kind to ssh username with private key
			- Username is username of github profile
				- On the system shell, do `ls ~/.ssh/` to see private key file. Then use `cat` to view `cat ~/.ssh/id_ed25519`
				- Copy the private key
			- Under private key, check 'enter directly'
			- Click add and then paste the key.
			- Enter ID and description
			- Then Add.
		+ Now select the defined credential from the dropdown//

	- Save

	- Specify branch to build

	- Now save.

4. Configure project on jenkins

- Go to the webhook project > configure
	- Under 'Build Rriggers', check 'Github hook trigger for GITScm polling'
	- Save

![git hook trig](https://github.com/uedwinc/Jenkins-Project/blob/main/images/git%20hook%20trig.png)

5. Now, add and push any new changes to the repository to confirm trigger on jenkins

![triggered]()

![trig complete]()

![slack succ]()