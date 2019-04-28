# Introduction
This is contain a small server, which act as our application for deployment given in this [repo](https://github.com/KTH-DD2480/smallest-java-ci). 

# Test application

## Dependencies to install
* Maven 4.0.0
* Java jre 8

## How to run the test app
To run this repo in Ubuntu -
* mvn package
* java -jar target/CI-jar-with-dependencies.jar

Then in your browser like Firefox type in http://localhost:8000/ to check that the server is reacting

For those who do not have Maven can still run this with java -jar target/CI-jar-with-dependencies.jar , the jar file is already precompiled in "target" folder

# Setup CI/CD pipeline instructions
These setup instructions should work for Ubuntu 18.04
## Install Jenkins 
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

# Set up Kubernetes dynamic slaves
## Plugins for Jenkins to install
* login to http://localhost:8080/
* Go to "Manage Jenkins"
* Scroll down and click on "Manage Plugins"
* Then click on the tab "Available" 
* Use the filter on the top right side to find and mark these following plugins
  * Google Compute Engine Plugin
  * Google Metadata plugin
  * Google OAuth Credentials plugin
If you can't find them, then they might be already installed( can find and check in Installed if they are already there)
## Install Google Cloud SDK
[Source](https://cloud.google.com/sdk/docs/downloads-apt-get) This is for Ubuntu 18.04
* export CLOUD_SDK_REPO="cloud-sdk-$(lsb_release -c -s)"
* echo "deb http://packages.cloud.google.com/apt $CLOUD_SDK_REPO main" | sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list
* curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
* sudo apt-get update && sudo apt-get install google-cloud-sdk

Or alternatively, this may work better for some by installing gcloud this way

* sudo apt-get install snapd
* sudo snap install google-cloud-sdk --classic

You can then setup your account with the command "gcloud init" then choose/create your project and also choose region(I chose 23 europe-west-2c, which is in London)

## Create a service account for Jenkins 
There are several ways to do this, but I find this way the easiest. 
* Go to [Google cloud console platform](https://console.cloud.google.com/?hl=sv)
* Then on the top left of the screen click on the ≡ mark and point at "IAM & admin" on the list. This will show another list and on that list you will click on "Service accounts"
* On that page click on "CREATE SERVICE ACCOUNT", service account name can be what ever you like(I chose my as "jkinsoutside") then click on "Create"
* On the next step as for role I chose "Owner" (If you prefer tighter security then you can choose some other roles), then "Continue"
* Now click on "Create key" and choose "JSON" format, then create and save it on your workstation. 
Now we should have a json keyfile for Jenkins to use for deploying and creating K8s slaves.

## Configure and setup Google cloud Engine plugin on Jenkins
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
# Monitoring setup (Prometheus & Grafana)
## Setup Prometheus
Note that this is a full proper setup for Prometheus on Ubuntu, so we are not satisfied with only getting it to run.

First we need to create usergroup so that we can use systemctl later for Prometheus to run in the background. 

* sudo useradd --no-create-home --shell /bin/false prometheus

Then we need to prepare 2 directories to store the files, which will be extracted from the .tar package download from Prometheus repo. Also we need to 

* sudo mkdir /etc/prometheus
* sudo mkdir /var/lib/prometheus

Also assign user to use these directories as prometheus user, which we already created in the first step.

* sudo mkdir /etc/prometheus
* sudo mkdir /var/lib/prometheus

Download .tar package and unpackage it.

* cd ~
* curl -LO https://github.com/prometheus/prometheus/releases/download/v2.0.0/prometheus-2.0.0.linux-amd64.tar.gz
* tar xvf prometheus-2.0.0.linux-amd64.tar.gz

Then copy the tool binary files and console library directories to the following directories

* sudo cp prometheus-2.0.0.linux-amd64/prometheus /usr/local/bin/
* sudo cp prometheus-2.0.0.linux-amd64/promtool /usr/local/bin/
* sudo cp -r prometheus-2.0.0.linux-amd64/consoles /etc/prometheus
* sudo cp -r prometheus-2.0.0.linux-amd64/console_libraries /etc/prometheus

Also set user ownership of these four as prometheus

* sudo chown prometheus:prometheus /usr/local/bin/prometheus
* sudo chown prometheus:prometheus /usr/local/bin/promtool
* sudo chown -R prometheus:prometheus /etc/prometheus/consoles
* sudo chown -R prometheus:prometheus /etc/prometheus/console_libraries

Now we have move all of the neccessary files for prometheus and now we only need to configure the prometheus.yml to initialize our scrape server. The file can be found in this repo so you can download it from here. Also we need to clean up the .tar and the unpacked directory, since we don't need it anymore.

* rm -rf prometheus-2.0.0.linux-amd64.tar.gz prometheus-2.0.0.linux-amd64

After downloading put the prometheus.yml into this directory /etc/prometheus/ . Alternatively you can manually write it

* sudo nano /etc/prometheus/prometheus.yml

Then copy and paste the content in prometheus.yml in this repo. Then again set user ownership of this yaml file as prometheus

* sudo chown prometheus:prometheus /etc/prometheus/prometheus.yml

Now we can initialize and run the server with the command below.

* sudo -u prometheus /usr/local/bin/prometheus \
    --config.file /etc/prometheus/prometheus.yml \
    --storage.tsdb.path /var/lib/prometheus/ \
    --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries

It should display at the end as 'level=info ts=2017-11-17T18:37:27.53890004Z caller=main.go:371 msg="Server is ready to receive requests."' or something like that. Now instead of just have the server occupy one of our terminal we can make so that it runs in the background. Exit with Ctr+c then put the file prometheus.service in this repo in the directory /etc/systemd/system/, or alternatively manually write it

* sudo nano /etc/systemd/system/prometheus.service

then copy paste the content from prometheus.service. Reload service files for the os system

* sudo systemctl daemon-reload

Now we can run it in the background by.

* sudo systemctl start prometheus

Status check to make sure that everything work. It should say that the server is active and running.

* sudo systemctl status prometheus

If you want prometheus to auto start on boot 

* sudo systemctl enable prometheus

Now you can visit your prometheus server at http://localhost:9090/ , note that port 9090 is because we specified it in the prometheus.yml file (see job_name: 'prometheus' in the file).

Next is to make the prometheus server to scrape data from the jenkins server. First is to add a new job in the prometheus.yml file, which it's actually already done as you can see in the same file we copied (see job_name: 'jenkins' in the prometheus.yml file). As specified in the file, Prometheus will automatically scrape data at the address "localhost:8080/prometheus", but before that happens we need to install the Prometheus plugin in our Jenkins server. 

* Go to "Manage Jenkins" from Jenkins dashboard -> "Manage Plugins" -> click on "Available" tab then search for "Prometheus metrics plugin" and install that. 
After installing this, visit http://localhost:9090/ then right beside the "Execute" button click on the radio button and choose something like "jenkins_runs_total_total" for displaying the total jenkins job run, then go back to jenkins and start one build one any pipeline you have. The number should then increase by one when you update the Prometheus server page.
## Setup Grafana
The installation is taken straight from [here](https://grafana.com/docs/installation/debian/)

* curl https://packages.grafana.com/gpg.key | sudo apt-key add -
* sudo apt-get update
* sudo apt-get install grafana

When this is done we can start the server by 

* systemctl start grafana-server

Then check status of the server, which should be active.

* systemctl status grafana-server

Now visit this server at http://localhost:3000/ then for first time login user: admin and password: admin . Then the server will request you to input a new password for future login. 

* On the dashboard page, look at the left and look for "Configuration"(look like a wheel), then click on "Data Sources"
* Now click on "Add data sources" and look for "Prometheus" then choose that.
* On the config page, "Name" should already be set as "Prometheus". "Url" should be "http://localhost:9090" and for "Access" choose "Browser". Then click Save and it should display "Data source is working"

Now look at the right again and find "Dashboards" and click on "Manage"

* On the "Manage" page click on "New Dashboard" and there should be already be a panel ready for you.
* On the panel click add query then write "jenkins_runs_total_total".
* After that go to "Visualization" and change radio button "Graph" to "Single stat" then on the "Value" section right below choose for "Stat" as "Current" instead of "Average". Then click at the top of the page "Save" to save the dashboard. 
You should now see the same number as the number you saw back at Prometheus server page.
# tools for the CI part (Maven & Github)
For the CI part we use ngrok to expose our Jenkins local server. How to install ngrok [please look here](https://ngrok.com/download)

* On Ubuntu 18.04 expose port by "./ngrok http 8080" after the installation assuming that the Jenkins port is at port 8080 by default

As for Maven installation is quite straight forward at this [link](https://maven.apache.org/download.cgi). Junit test suite is already included as a dependency in the pom file for this repo.

# tools for the CD part (Gcloud local builder & Gcloud registry)
Assuming that you have already installed Google Cloud SDK in the previous sections, here we will install a tool component call Google cloud local builder. Open bash terminal then 

* gcloud components install cloud-build-local

After the installation is done you can go ahead and visit [Gcloud registry](https://cloud.google.com/container-registry/) and click on view console if you have already logged into your Google account. Then enable the GCR(Google Cloud Registry) if you have not enable it yet. It's required to use GCR in order to deploy with Kubernetes. To build and push into GCR using Gcloud local builder the syntax is 

* "gcloud builds submit --tag gcr.io/${projectname}/${registryreponame:version} ."

${projectname} is your ProjectID and it can be checked with "gcloud init" command in bash shell. ${registryreponame:version} can be whatever you think appropriate. For instance it can be gcr.io/project-1234/appdeploy:v1  . Do note that the dot at the end is not a typo(the syntax look almost the same as docker push). If the push is successful, you'll see it at [Gcloud registry](https://cloud.google.com/container-registry/) (you might need to switch project to the projectid you have pushed to).

## Jenkins declarative pipeline for auto pushing and deploy 
To be able to run kubectl and gcloud shell command in Jenkinsfile for declarative pipeline, we need to authenticate itself before for instance running the command above for pushing to GCR . The easiest way to authenticate Jenkins outside the cluster is to use the service account .json file we got from the previous section for Jenkins to authenticate itself so that it can use kubectl and gcloud. There are 2 important things to find, first is the absolute path to the binary file gcloud, which is usually located in /home/${user}/google-cloud-sdk/bin , otherwise please look for the directory "google-cloud-sdk" and check its absolute path. There is a reason for thing to be tedious as this because the home directory for Jenkins is in /var/lib/jenkins by default, which is different than our own home directory, therefore it can't find the files meant for us to use. Second thing is the absolute path to the authentication .json file you created previously for service account. Now Jenkins is able to authenticate itself with these shell command below. Preferably it would look nicer with a variable defining the path

* withEnv(['GCLOUD_PATH=/home/${user}/google-cloud-sdk/bin']) {....}
* in the {....} write "$GCLOUD_PATH/gcloud auth activate service-account --key-file=PATH_TO_JSONFILE"
* then new line "$GCLOUD_PATH/gcloud container clusters get-credentials ${name_of_yourcluster} --zone ${your_zone} --project ${your_project_id}" 
* new line again "$GCLOUD_PATH/gcloud config set project ${your_project_id}" 

Zone and project id can be found with "gcloud init" while cluster with "gcloud container clusters list" assuming that you created your cluster with "gcloud container clusters create --num-nodes= " and look for your cluster which you want to deploy to. Now we can push using gcloud and use kubectl to deploy as shell commands with Jenkins like how we do it manually on the terminal. Also the Jenkinsfile for my pipeline is provided in this repo. You can doublecheck these steps with it. 



