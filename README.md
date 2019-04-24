## Introduction
This is contain a small server, which act as our application for deployment given in this [repo](https://github.com/KTH-DD2480/smallest-java-ci). 
## Dependencies to install
* Maven 4.0.0
* Java jre 8

## How to run
To run this repo in Ubuntu -
* mvn package
* java -jar target/CI-jar-with-dependencies.jar

Then in your browser like Firefox type in http://localhost:8000/ to check that the server is reacting

For those who do not have Maven can still run this with java -jar target/CI-jar-with-dependencies.jar , the jar file is already precompiled in "target" folder

## Setup CI/CD pipeline instructions
These setup instructions should work for Ubuntu 18.04
# Install Jenkins 
To run Jenkins we need java 8 so it must be installed first
* sudo apt update
* sudo apt install openjdk-8-jdk
Then we need to add a GPG key of the Jenkins repo and then apt install it
* wget -q -O - https://pkg.jenkins.io/debian/jenkins.io.key | sudo apt-key add -
* sudo sh -c 'echo deb http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
* sudo apt update
* sudo apt install jenkins
Double check with
* systemctl status jenkins
Jenkins server should be active by now. By default the server should be at local port 8080, so in your browser connect to http://localhost:8080/ . On that page Jenkins server will then demand the password from you and it will also tell you where the password file is in your system. My password file is at /var/lib/jenkins/secrets/initialAdminPassword, so
* sudo cat /var/lib/jenkins/secrets/initialAdminPassword
Then choose "Install suggested plugins" and you should get a basic working Jenkins server.

## Set up Kubernetes dynamic slaves
# Plugins for Jenkins to install
* login to http://localhost:8080/
* Go to "Manage Jenkins"
* Scroll down and click on "Manage Plugins"
* Then click on the tab "Available" 
* Use the filter on the top right side to find and mark these following plugins
  * Google Compute Engine Plugin
  * Google Metadata plugin
  * Google OAuth Credentials plugin
If you can't find them, then they might be already installed( can find and check in Installed if they are already there)
# Install Google Cloud SDK
[Source](https://cloud.google.com/sdk/docs/downloads-apt-get) This is for Ubuntu 18.04
* export CLOUD_SDK_REPO="cloud-sdk-$(lsb_release -c -s)"
* echo "deb http://packages.cloud.google.com/apt $CLOUD_SDK_REPO main" | sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list
* curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
* sudo apt-get update && sudo apt-get install google-cloud-sdk
You can then setup your account with the command "gcloud init" then choose/create your project and also choose region(I chose 23 europe-west-2c, which is in London)

# Create a service account for Jenkins 
There are several ways to do this, but I find this way the easiest. 
* Go to [Google cloud console platform](https://console.cloud.google.com/?hl=sv)
* Then on the top left of the screen click on the ≡ mark and point at "IAM & admin" on the list. This will show another list and on that list you will click on "Service accounts"
* On that page click on "CREATE SERVICE ACCOUNT", service account name can be what ever you like(I chose my as "jkinsoutside") then click on "Create"
* On the next step as for role I chose "Owner" (If you prefer tighter security then you can choose some other roles), then "Continue"
* Now click on "Create key" and choose "JSON" format, then create and save it on your workstation. 
Now we should have a json keyfile for Jenkins to use for deploying and creating K8s slaves.

# Configure and setup Google cloud Engine plugin
