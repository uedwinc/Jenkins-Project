# Jenkins Project

This project will incorporate:
- Using Maven to build a pom.xml into a '.war' app file
- Pushing the artifact into Nexus repository
- Integrating Sonarqube for code analysis and setting up quality gate
- Building a docker image for our application
- Pushing the image to ECR
- Deploying the image with ECS
- Configuring a Jenkins-Github webhook to build at push
- Integrating Slack notification

## Installations

**Starting and Configuring the Jenkins Server**

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

**Starting and Configuring Nexus Server**

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

**Starting and Configuring Sonarqube Server**

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

## Configure Sonarqube Analysis

1. Under "Manage Jenkins", go to "Tools"

    - Go to "SonarQube Scanner" section and add sonarqube scanner
	    - Name eg sonar5.0.1
	    - Select the appropriate version.
	    - Save.

![sonar5]()

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