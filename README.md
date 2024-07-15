# Develop your Google Cloud Network 

## Challenge scenario
As a cloud engineer at Jooli Inc. and recently trained with Google Cloud and Kubernetes, you have been asked to help a new team (Griffin) set up their environment. The team has asked for your help and has done some work, but needs you to complete the work.

You are expected to have the skills and knowledge for these tasks so don’t expect step-by-step guides.

You need to complete the following tasks:

  * Create a development VPC with three subnets manually
  * Create a production VPC with three subnets manually
  * Create a bastion that is connected to both VPCs
  * Create a development Cloud SQL Instance and connect and prepare the WordPress environment
  * Create a Kubernetes cluster in the development VPC for WordPress
  * Prepare the Kubernetes cluster for the WordPress environment
  * Create a WordPress deployment using the supplied configuration
  * Enable monitoring of the cluster
  * Provide access for an additional engineer
  
Some Jooli Inc. standards you should follow:

  * Create all resources in the **REGION** region and **ZONE** zone, unless otherwise directed.
  * Use the project VPCs.
  * Naming is normally team-resource, e.g. an instance could be named **kraken-webserver1**.
  * Allocate cost effective resource sizes. Projects are monitored and excessive resource use will result in the containing project's termination (and possibly yours), so beware. This is the guidance the monitoring team is willing to share: unless directed, use **e2-medium**.

![image](https://github.com/user-attachments/assets/9df05bbb-3dca-40b7-a1d3-35854e38e7ac)

### Task 1. Create development VPC manually
Create a VPC called griffin-dev-vpc with the following subnets only:
  
    griffin-dev-wp
      IP address block: 192.168.16.0/20
    griffin-dev-mgmt
      IP address block: 192.168.32.0/20
<details>
<summary>Task 1 Answer</summary>
<br>
 
Run the below commands in Cloud Shell:

```
gcloud compute networks create griffin-dev-vpc --subnet-mode custom
 
gcloud compute networks subnets create griffin-dev-wp --network=griffin-dev-vpc --region us-east1 --range=192.168.16.0/20
 
gcloud compute networks subnets create griffin-dev-mgmt --network=griffin-dev-vpc --region us-east1 --range=192.168.32.0/20
```
</details>

### Task 2. Create production VPC manually
Create a VPC called griffin-prod-vpc with the following subnets only:
    
    griffin-prod-wp
      IP address block: 192.168.48.0/20
    griffin-prod-mgmt
      IP address block: 192.168.64.0/20

<details>
<summary>Task 2 Answer</summary>
<br>
 
Run the below commands in Cloud Shell:

```
sutil cp -r gs://cloud-training/gsp321/dm .
 
cd dm
 
sed -i s/SET_REGION/us-east1/g prod-network.yaml
 
gcloud deployment-manager deployments create prod-network \
  --config=prod-network.yaml
 
cd ..
```
</details>

### Task 3. Create bastion host
Create a bastion host with two network interfaces, one connected to **griffin-dev-mgmt** and the other connected to **griffin-prod-mgmt**. Make sure you can SSH to the host. 


<details>
<summary>Task 3 Answer</summary>
<br>
 
Run the below commands in Cloud Shell:

```
gcloud compute instances create bastion --network-interface=network=griffin-dev-vpc,subnet=griffin-dev-mgmt  --network-interface=network=griffin-prod-vpc,subnet=griffin-prod-mgmt --tags=ssh --zone=us-east1-b
 
gcloud compute firewall-rules create fw-ssh-dev --source-ranges=0.0.0.0/0 --target-tags ssh --allow=tcp:22 --network=griffin-dev-vpc
 
gcloud compute firewall-rules create fw-ssh-prod --source-ranges=0.0.0.0/0 --target-tags ssh --allow=tcp:22 --network=griffin-prod-vpc
```
</details>


### Task 4. Create and configure Cloud SQL Instance
Create a MySQL Cloud SQL Instance called **griffin-dev-db** in *REGION*.
Connect to the instance and run the following SQL commands to prepare the WordPress environment:
  
    CREATE DATABASE wordpress;
    CREATE USER "wp_user"@"%" IDENTIFIED BY "stormwind_rules";
    GRANT ALL PRIVILEGES ON wordpress.* TO "wp_user"@"%";
    FLUSH PRIVILEGES;

These SQL statements create the worpdress database and create a user with access to the wordpress database.

You will use the username and password in task 6.

<details>
<summary>Task 4 Answer</summary>
<br>
 
* In Google Cloud Console, go to SQL -> Create Instance -> MySQL
* Type Instance ID as given in Task 4 lab instructions, and for Password enter any password and save it on a notepad to be used later.
* For Database version, choose MySQL 5.7, and for Region choose “us-east-1” as given in lab instructions. Click Create Instance. Wait for sometime (approx. 5-7 minutes) till the instance gets created.
* In the SQL dashboard, click on Open CloudShell. A command is automatically entered in the terminal. Press Enter to run that command. When prompted for ‘Password’, use the password which was set while creating the MySQL instance. You will enter MySQL interface.
* In the MySQL interface, run the below command (as given in Task 4 instructions):

```
 CREATE DATABASE wordpress;
 CREATE USER "wp_user"@"%" IDENTIFIED BY "stormwind_rules";
 GRANT ALL PRIVILEGES ON wordpress.* TO "wp_user"@"%";
 FLUSH PRIVILEGES;
```
Enter ‘exit’ to exit out of MySQL interface.

</details>


### Task 5. Create Kubernetes cluster
Create a 2 node cluster (e2-standard-4) called **griffin-dev**, in the **griffin-dev-wp** subnet, and in zone *ZONE*.

<details>
<summary>Task 5 Answer</summary>
 <br>
 
Run the below commands in Cloud Shell:
```
gcloud container clusters create griffin-dev \
--network griffin-dev-vpc \
--subnetwork griffin-dev-wp \
--machine-type n1-standard-4 \
--num-nodes 2  \
--zone us-east1-b
 
gcloud container clusters get-credentials griffin-dev --zone us-east1-b
 
cd ~/
 
gsutil cp -r gs://cloud-training/gsp321/wp-k8s .
```
</details>

### Task 6. Prepare the Kubernetes cluster
 * From Cloud Shell copy all files from **gs://cloud-training/gsp321/wp-k8s**.
The WordPress server needs to access the MySQL database using the username and password you created in task 4.

 * You do this by setting the values as secrets. WordPress also needs to store its working files outside the container, so you need to create a volume.

 * Add the following secrets and volume to the cluster using **wp-env.yaml**.

 * Make sure you configure the *username* to **wp_user** and *password* to **stormwind_rules** before creating the configuration.

You also need to provide a key for a service account that was already set up. This service account provides access to the database for a sidecar container.

 * Use the command below to create the key, and then add the key to the Kubernetes environment:

``` 
  gcloud iam service-accounts keys create key.json \
      --iam-account=cloud-sql-proxy@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
  kubectl create secret generic cloudsql-instance-credentials \
      --from-file key.json
```
   

<details>
<summary>Task 6 Answer</summary>
<br>
 
* In Cloud Shell, click Open Editor. Click Open in new window if prompted.
* Expand wp-k8s -> wp-env.yaml
* In the file, change username and password as given in Task 6 lab instructions. Click File -> Save.
* Run the below commands in Cloud Shell:

```
 cd wp-k8s

 kubectl create -f wp-env.yaml

 gcloud iam service-accounts keys create key.json \
   --iam-account=cloud-sql-proxy@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
 kubectl create secret generic cloudsql-instance-credentials \
   --from-file key.json
```
</details>

### Task 7. Create a WordPress deployment
Now that you have provisioned the MySQL database, and set up the secrets and volume, you can create the deployment using **wp-deployment.yaml**.

 * Before you create the deployment you need to edit **wp-deployment.yaml**.

 * Replace **YOUR_SQL_INSTANCE** with griffin-dev-db's **Instance connection name**.

 * Get the **Instance connection name** from your Cloud SQL instance.

 * After you create your WordPress deployment, create the service with **wp-service.yaml**.

 * Once the Load Balancer is created, you can visit the site and ensure you see the WordPress site installer.

At this point the dev team will take over and complete the install and you move on to the next task.

![image](https://github.com/user-attachments/assets/1829d478-55ff-4456-86c7-505448be1fb8)

<details>
<summary>Task 7 Answer</summary>

 <br>
 
* In Editor, expand wp-k8s -> wp-deployment.yaml
* Replace YOUR_SQL_INSTANCE with griffin-dev-db's Instance connection name. Click File -> Save.
* Run the below commands in Cloud Shell:

```
kubectl create -f wp-deployment.yaml
kubectl create -f wp-service.yaml
```
</details>


### Task 8. Enable monitoring
Create an uptime check for your WordPress development site.

<details>
<summary>Task 8 Answer</summary>
 <br>
 
 * In Google Cloud Console, go to Kubernetes Engine -> Services and Ingress ->
 * Copy endpoint's (wordpress) IP address.
 * In Google Cloud Console, go to Monitoring -> Uptime Checks -> + CREATE UPTIME CHECK.
 * Fill in the details as provided below:

```
   Target: Hostname : endpoint's IP address (without http and port number)
   Path : /
   Title: Wordpress Uptime
```
   
 * Leave everything as default. Click Next -> Next -> Create
</details>


### Task 9. Provide access for an additional engineer
You have an additional engineer starting and you want to ensure they have access to the project. Grant them the editor role to the project.
The second user account for the lab represents the additional engineer.

<details>
<summary>Task 9 Answer</summary>
 <br>
 
 * In Google Cloud Console, go to IAM & Admin -> IAM -> Grant Access
 * For New Principals: copy Username 2 from the left panel on lab instructions page
 * For Select a role,  Basic -> Editor. Click Save.
   
</details>

## Sample Answer
Checkout the collapsed section for each task!! :heart_eyes_cat:

