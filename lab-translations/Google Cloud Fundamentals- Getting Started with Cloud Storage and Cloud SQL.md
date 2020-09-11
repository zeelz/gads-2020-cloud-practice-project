# Google Cloud Fundamentals: Getting Started with Cloud Storage and Cloud SQL

## Task 1: Sign in to the Google Cloud Platform (GCP) Console

Sign in with the Qwiklab provided credentials.
On the GCP Console top-right menu, click ***Activate Cloud Shell*** icon to open ***Cloud Shell***. When prompted, click ***Continue*** to proceed.

*Since we would be using only Cloud Shell for this lab you might as well click the **Open in new window** icon at the top right of Cloud Shell window to open it in a new tab.*

## Task 2: Deploy a web server VM instance

### Create Compute Instance with a startup script

*To enable external IP for public access add **http-server** tag created by the default firewall rules to allow http*

```
gcloud compute instances create bloghost \
--zone us-central1-a \
--machine-type n1-standard-1 \
--image=debian-9-stretch-v20200805 \
--image-project=debian-cloud \
--tags=http-server \
--metadata startup-script="apt-get update; apt-get install apache2 php php-mysql -y; service apache2 restart"
```


## Task 3: Create a Cloud Storage bucket

*All Cloud Storage bucket names must be globally unique. To ensure that your bucket name is unique, these instructions will guide you to give your bucket the same name as your Cloud Platform project ID, which is also globally unique.*

* To create a multi-region bucket export your chosen location to an environment variable called `LOCATION`.
Multi-regions are ASIA, EU and US

`export LOCATION=US`


* In Cloud Shell, the DEVSHELL_PROJECT_ID environment variable contains your project ID. Use it alongside the LOCATION variable to create a bucket named after your project ID


`gsutil mb -l $LOCATION gs://$DEVSHELL_PROJECT_ID`


* Retrieve a banner image from a publicly accessible Cloud Storage location:

`gsutil cp gs://cloud-training/gcpfci/my-excellent-blog.png my-excellent-blog.png`

* Copy the banner image to your newly created Cloud Storage bucket:

`gsutil cp my-excellent-blog.png gs://$DEVSHELL_PROJECT_ID/my-excellent-blog.png`

* Modify the Access Control List of the object you just created so that it is readable by everyone:

`gsutil acl ch -u allUsers:R gs://$DEVSHELL_PROJECT_ID/my-excellent-blog.png`


## Task 4: Create the Cloud SQL instance

* Enter the following command to create a Cloud SQL instance with **blog-db** as instance id and your chosen password as PASSWORD in zone us-central1-a (as provided as by lab)

`gcloud sql instances create blog-db --zone us-central1-a --root-password PASSWORD`

* To view the SQL instance's public IP address, which you will need for later use, run:

`gcloud sql instances describe blog-db`

 
* Create a new database user with blogdbuser as username and a password of your choice

```
gcloud sql users create blogdbuser \
--host % \
--instance blog-db
--password PASSWORD
```

* Add a network range to authorized compute instance access to Cloud SQL instance.

To allow **bloghost** access to this SQL instance grab its external IP and add it as a CIDR address *(append /32 to the end of the IP)*

To view **bloghost**'s external IP run:

`gcloud compute instances list | grep bloghost`

```
gcloud sql instances patch blog-db \
--authorized-networks VM_INSTANCE_IP/32
```


## Task 5: Configure an application in a Compute Engine instance to use Cloud SQL

* SSH into **bloghost**

`gcloud compute ssh bloghost --zone us-central1-a`

* Change your working directory to the document root of the web server:

`cd /var/www/html`


* Script to paste:

```
<html>
<head><title>Welcome to my excellent blog</title></head>
<body>
<h1>Welcome to my excellent blog</h1>
<?php
 $dbserver = "CLOUDSQLIP";
$dbuser = "blogdbuser";
$dbpassword = "DBPASSWORD";
// In a production blog, we would not store the MySQL
// password in the document root. Instead, we would store it in a
// configuration file elsewhere on the web server VM instance.

$conn = new mysqli($dbserver, $dbuser, $dbpassword);

if (mysqli_connect_error()) {
        echo ("Database connection failed: " . mysqli_connect_error());
} else {
        echo ("Database connection succeeded.");
}
?>
</body></html>
```

* Use the nano text editor to edit a file called index.php and paste the above script:

`sudo nano index.php`

* Press Ctrl+O, and then press Enter to save your edited file.
* Press Ctrl+X to exit the nano text editor.
* Restart the web server:

`sudo service apache2 restart`

* Preview the just created web file by opening a new browser tab and enter **bloghost**'s external IP address, append /index.php

*When you load the page, you will see that its content includes an error message beginning with the words:*

***Database connection failed: ...***

*This message occurs because you have not yet configured PHP's connection to your Cloud SQL instance.*

* Return to your ssh session on bloghost. Use the nano text editor to edit index.php again.

`sudo nano index.php`

Replace CLOUDSQLIP with the Cloud SQL instance Public IP address and DBPASSWORD with the Cloud SQL database password

* Press Ctrl+O, and then press Enter to save your edited file.
* Press Ctrl+X to exit the nano text editor.
* Restart the web server:

`sudo service apache2 restart`

Type `exit` and press ENTER to return to your Cloud Shell instance

* Return to the web browser tab in which you opened your bloghost VM instance's external IP address. When you load the page, the following message appears:

***Database connection succeeded.***


## Task 6: Configure an application in a Compute Engine instance to use a Cloud Storage object

*Recall we copied an image from a publicly accessible bucket into our own bucket. We will now insert it in our **index.php** file using its public URL since we made it publicly accessible.*

Publicly accessible images have the following URL format:

`https://storage.googleapis.com/BUCKET_NAME/IMAGE_NAME`

In our case, the BUCKET_NAME is your project ID and IMAGE_NAME is my-excellent-blog.png. We will refer to it as IMAGE_PUBLIC_URL

* Return to your ssh session on your **bloghost** VM instance or reconnect if it was exited.

`gcloud compute ssh bloghost --zone us-central1-a`

* Navigate to the web folder and use the nano text editor to edit index.php again.

`cd /var/www/html`

`sudo nano var/www/html/index.php`

* Use the arrow keys to move the cursor to the line that contains the **h1** element. Press Enter to open up a new, blank screen line, and the following HTML code:

`<img src='IMAGE_PUBLIC_URL'>`

The resulting line will look like this:

`<img src='https://storage.googleapis.com/qwiklabs-gcp-0005e186fa559a09/my-excellent-blog.png'>`

* Press Ctrl+O, and then press Enter to save your edited file.
* Press Ctrl+X to exit the nano text editor.
* Restart the web server:

`sudo service apache2 restart`

* Return to the web browser tab in which you opened your bloghost VM instance's external IP address. When you load the page, its content now includes a banner image.

## Congratulations!

In this lab, you configured a Cloud SQL instance and connected an application in a Compute Engine instance to it. You also worked with a Cloud Storage bucket.