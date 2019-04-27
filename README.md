## Introduction
This is contain a small server, which act as our application for deployment given in this [repo](https://github.com/KTH-DD2480/smallest-java-ci). 
## Test application
# Dependencies to install
* Maven 4.0.0
* Java jre 8

# How to run the test app
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

# Configure and setup Google cloud Engine plugin on Jenkins
Some vital informations about setting up this plugin are [here](https://cloud.google.com/solutions/using-jenkins-for-distributed-builds-on-compute-engine) 
We need to create a disk image as template for our slave VM and minimum this image must have java 8 installed. This can be found at this [page](https://cloud.google.com/solutions/using-jenkins-for-distributed-builds-on-compute-engine), at subsection "Create a jenkins image", but I'll show how I did it anyway.
* Go to bash terminal and install unpacker 
  * wget https://releases.hashicorp.com/packer/0.12.3/packer_0.12.3_linux_amd64.zip
  * unzip packer_0.12.3_linux_amd64.zip
* Find out the project name 
  * export PROJECT=$(gcloud info --format='value(config.project)')
* Then run this to create a json configuration file for our image 
cat > jenkins-agent.json <<EOF
{
  "builders": [
    {
      "type": "googlecompute",
      "project_id": "$PROJECT",
      "source_image_family": "ubuntu-1604-lts",
      "source_image_project_id": "ubuntu-os-cloud",
      "zone": "us-central1-a",
      "disk_size": "10",
      "image_name": "jenkins-agent-{{timestamp}}",
      "image_family": "jenkins-agent",
      "ssh_username": "ubuntu"
    }
  ],
  "provisioners": [
    {
      "type": "shell",
      "inline": ["sudo apt-get update",
                  "sudo apt-get install -y default-jdk"]
    }
  ]
}
EOF
* Finally ./packer build jenkins-agent.json
This should show the image name created like this "googlecompute: A disk image was created: jenkins-agent-1494277483"

Now to begin configuring the plugin on jenkins
* login to http://localhost:8080/
* Go to "Manage Jenkins"
* Go to "Configure System"
* Scroll down all the way to the bottom and click on "Add a new cloud", then choose Google Compute Engine
* On the Name I wrote "gce", but it can be whatever you like.
* On Project ID , you can find it by for instance "gcloud init" in bash terminal, then look for your project id you have set for your current logged-in account 
* On Instance Cap you set how many slaves Jenkins can create maximum on GKE. For this I chose 1 just for demonstration purpose.
* Service Account Credentials , choose "Add" then "Jenkins" and then At "Kind" look for Google Service Account from private key
  * Project Name can be what ever you like (I chose "jenkins-gcekey")
  * Choose JSON key then browse for the .json file we got from the previous section we downloaded when the service account was created. 
  * Click Add at the bottom
  * Now we are back at "Service Account Credentials" again click on the radio button left to "Add" and look for the Project Name you just used to load the crediential .json. It should says after clicking somewhere else "The credential successfully made an API request to Google Compute Engine" right below the "Add" button.
* At instance configuration Click add and now there are more to configure
* At Name Prefix choose whatever you like(it's the name prefix of the VM created on K8s), I call my gceslave0
* Description can also be whatever you like
* Launch TimeOut is the time before the slave should be automatically be deleted and it's in seconds. I chose 60 seconds.
* Node Retention time and Usage should be left as it is
* Labels , this is used for the descriptive pipeline when we call our agents so name it something good. I call my "gceslave0"
* Now other things should be left as they are. Instead look for "Location" Region set your project region(preferable the same as when you set up with gcloud init, my was 23 so it should be europe-west2) and Zone (23 also means europe-west2-c for my zone).
* Ignore "Template to use" and click on Advanced
* Machine Type , I chose n1-standard-1
* Minimum Cpu Plattform , Intel Skylake
* As for "Networking" choose default for both network and Subnetwork
* Also mark the box for "Attach External IP?"
* At "Boot Disk" section choose image project as your project id(can be found in gcloud init) and image name is the name of the image created like "jenkins-agent-1494277483" should be shown on the list.
* Disk Type I chose pd-standard since ssd cost more money. Then click "Save" and we are done with the configuration.
Now this should be working and we can check that by 
* From Jenkins dashboard go to "Manage Jenkins" 
* Then scroll down and click on "Manage nodes" 
* Then you should see a radio button "Provision via gce" click on that and you should see your description which you have written previously during the configuration. 
* Click on that and it should take you to a new page(page for the slave instance), check on the left and click on "Log" to see the setup procedure for the node. If the logs will ends after with "Agent successfully connected and online". Don't worry if you see something like "Failed to connect via ssh..", because it takes sometime before the VM on GCE to be completely set up and after some tries it should say "connected via SSH".
Now we can write a pipeline script for running stuffs at the slave.

