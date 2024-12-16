# sonar

**Step 1: Launch an Ubuntu EC2 Instance**
Go to the AWS Management Console and launch a new EC2 instance.

Choose Ubuntu as the AMI (Amazon Machine Image).
Choose the instance type (e.g., t2.micro for testing).
Set up the security group (open ports 22 for SSH and 9000 for SonarQube).
Create a key pair to access the instance and download the key file.
Access the EC2 instance:

Use SSH to connect to your EC2 instance:
bash
Copy code
ssh -i your-key-file.pem ubuntu@<EC2-PUBLIC-IP>
Step 2: Install SonarQube on EC2
Update package lists:

bash
Copy code
sudo apt update
Install Java (SonarQube requires Java 11 or later):

bash
Copy code
sudo apt install openjdk-11-jdk -y
Download SonarQube:

Go to the SonarQube Downloads page to get the latest version link.
Example:
bash
Copy code
wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.7.0.61597.zip
Install unzip utility (if not installed):

bash
Copy code
sudo apt install unzip
Extract SonarQube:

bash
Copy code
sudo unzip sonarqube-9.7.0.61597.zip -d /opt
Configure SonarQube to run as a service:

Create a sonar user for security reasons:

bash
Copy code
sudo useradd sonar
sudo chown -R sonar:sonar /opt/sonarqube
Start SonarQube:

bash
Copy code
sudo /opt/sonarqube/bin/linux-x86-64/sonar.sh start
Access SonarQube:

Open a browser and go to:
vbnet
Copy code
http://<EC2-PUBLIC-IP>:9000
Default credentials:
Username: admin
Password: admin
Step 3: Install and Configure Sonar Scanner
Install Sonar Scanner:

Go to Sonar Scanner Download Page for the latest version.
Example:
bash
Copy code
wget https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-4.7.0.2747-linux.zip
sudo unzip sonar-scanner-4.7.0.2747-linux.zip -d /opt
Set Environment Variables:

bash
Copy code
echo "export SONAR_SCANNER_HOME=/opt/sonar-scanner-4.7.0.2747-linux" >> ~/.bashrc
echo "export PATH=$PATH:/opt/sonar-scanner-4.7.0.2747-linux/bin" >> ~/.bashrc
source ~/.bashrc
Step 4: Integrate SonarQube with GitHub
Create a GitHub App:

Go to GitHub Apps and click on New GitHub App.
Fill out the necessary details and select the required permissions for your repository.
After creating the app, note down the App ID, Client ID, and Client Secret.
Configure SonarQube to Connect with GitHub:

In the SonarQube dashboard (http://<EC2-PUBLIC-IP>:9000), go to Administration > General Settings > GitHub Integration.
Enter your App ID, Client ID, and Client Secret.
Install the GitHub App on your repositories:

Go to the GitHub App page and install the app on your desired repositories.
Step 5: Clone Your GitHub Repository and Run Sonar Scanner
Clone the Repository to your EC2 instance:

bash
Copy code
git clone <your-github-repo-url>
cd <repository-folder>
Configure SonarQube in the Repository:

Create a file named sonar-project.properties in the root of your project with the following content:
properties
Copy code
sonar.projectKey=your-project-key
sonar.projectName=Your Project Name
sonar.projectVersion=1.0
sonar.sources=.
sonar.host.url=http://<EC2-PUBLIC-IP>:9000
sonar.login=<your-sonarqube-token>
Run Sonar Scanner:

Execute the following command inside your project directory:
bash
Copy code
sonar-scanner
