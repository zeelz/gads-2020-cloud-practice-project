# Implement Private Google Access and Cloud NAT

## Sign in to the Google Cloud Platform (GCP) Console

Sign in with the Qwiklab provided credentials.
On the GCP Console top-right menu, click ***Activate Cloud Shell*** icon to open ***Cloud Shell***. When prompted, click ***Continue*** to proceed.

## Task 1. Create the VM instance

### Create a VPC network and subnet
`gcloud compute networks create privatenet --subnet-mode=custom --bgp-routing-mode=regional`

`gcloud compute networks subnets create privatenet-us --range=10.130.0.0/20 --network=privatenet --region=us-central1`

### Create firewall rules
```
gcloud compute firewall-rules create privatenet-allow-ssh \
--direction=INGRESS \
--priority=1000 \
--network=privatenet \
--action=ALLOW \
--rules=tcp:22 \
--source-ranges=35.235.240.0/20
```

### Create the VM instance with no public IP address
```
gcloud compute instances create vm-internal \
--zone us-central1-c \
--subnet privatenet-us \
--machine-type n1-standard-1 \
--no-address
```

### SSH into *vm-internal* to test the IAP tunnel
`gcloud compute ssh vm-internal --zone us-central1-c --tunnel-through-iap`

If prompted, type Y to continue.
When prompted for a passphrase, press ENTER.
When prompted for the same passphrase, press ENTER.

To test the external connectivity of **vm-internal**, run the following command

`ping -c 2 www.google.com`

This should not work because **vm-internal** has no external IP address!

Wait for the ping command to complete

Type `exit` and press ENTER to return to your Cloud Shell instance


## Task 2. Enable Private Google Access

### Create a Cloud Storage Bucket
*Use unique names for Bucket names such as your project id*

We will refer to your bucket name as *BUCKET_NAME*

Create multi-regional bucket in US region
 
`gsutil mb -l US gs://BUCKET_NAME`


### Copy an image file into your bucket
`gsutil cp gs://cloud-training/gcpnet/private/access.svg gs://BUCKET_NAME`

Confirm the image file was copied by viewing the bucket

`gsutil ls gs://BUCKET_NAME`

### Access the image from your VM instance

#### Copy image to Cloud Shell from bucket

Enter the following command to copy the image from your bucket to current directory

`gsutil cp gs://BUCKET_NAME/*.svg .`

Type `ls` to confirm the *svg* file is listed in your current direction

To connect to **vm-internal**, run the following command:

`gcloud compute ssh vm-internal --zone us-central1-c --tunnel-through-iap`

If prompted, type Y and press ENTER to continue


#### Copy image to VM instance from Bucket
Run the same command to copy the file to the instance's home directory
`gsutil cp gs://BUCKET_NAME/*.svg .`

This should not work: **vm-internal** can only send traffic within the VPC network because Private Google Access is disabled by default.

Press `Ctrl+C` to stop the request.

Enter `exit` to return to your Cloud Shell instance

### Enable Private Google Access
```
gcloud compute networks subnets update privatenet-us \
--region us-central1 \
--enable-private-ip-google-access
```

Now that we have enabled Private Google Access let us try to Copy the Bucket image to **vm-internal** again

`gcloud compute ssh vm-internal --zone us-central1-c --tunnel-through-iap`

If prompted, type Y and press ENTER to continue

`gsutil cp gs://BUCKET_NAME/*.svg .`

You should see *"operation completed ..."* output

Enter `exit` to return to your Cloud Shell instance

## Task 3. Configure a Cloud NAT gateway

### Try to update the VM instances
In Cloud Shell, to try to re-synchronize the package index

`sudo apt-get update`

This should work because Cloud Shell has an external IP address and the output should finish like this:

```
...
Reading package lists... Done

```

### To connect to vm-internal, and try the same command to re-synchronize the package index:
`gcloud compute ssh vm-internal --zone us-central1-c --tunnel-through-iap`

`sudo apt-get update`

Only Google Cloud packages will be updated in this case because **vm-internal** only has access to Google APIs and services, since Private Google Access was enabled in the subnet vm-internal belongs to.

Press `Ctrl+C` to stop and enter `exit` to return to your Cloud Shell instance

### Configure a Cloud NAT gateway
*Cloud NAT is a regional resource. You can configure it to allow traffic from all ranges of all subnets in a region, from specific subnets in the region only, or from specific primary and secondary CIDR ranges only.*

#### Create a Cloud Router to setup a new Cloud NAT Configuration

```
gcloud compute routers create nat-router \
--region=us-central1 \
--network=privatenet
```

You should see the following output with a listing of the router

*Creating router [nat-router]...done.* 

#### Create a new Cloud NAT Configuration with the newly created

```
gcloud compute routers nats create nat-config \
--router=nat-router \
--auto-allocate-nat-external-ips \
--nat-all-subnet-ip-ranges \
--region=us-central1
```

***Creating NAT [nat-config] in router [nat-router]...done.***

# keme explain those parameters

Again, let us try to re-synchronize the package index for **vm-internal**. SSH into **vm-internal** and type:

`sudo apt-get update`

This should work now because **vm-internal** is using the NAT gateway!
```
...
Reading package lists... Done
```

Enter `exit` to return to your Cloud Shell instance

*The Cloud NAT gateway implements outbound NAT, but not inbound NAT. In other words, hosts outside of your VPC network can only respond to connections initiated by your instances; they cannot initiate their own, new connections to your instances via NAT.*


## Task 4. Configure and view logs with Cloud NAT Logging

*Cloud NAT logging allows you to log NAT connections and errors. When Cloud NAT logging is enabled, one log entry can be generated for each of the following scenarios:*

* When a network connection using NAT is created.
* When a packet is dropped because no port was available for NAT.

### Enabling logging for both Translations and errors

```
gcloud compute routers nats update nat-config \
--router=nat-router \
--region=us-central1 \
--enable-logging
```

### NAT logging in Cloud Operations
Try to view logs by running the command:

```
gcloud logging read 'resource.type=nat_gateway' \
--limit=10 
--format=json
```

Cloud NAT logging has been setup but no logs were returned since no logging scenario has occurred

### Let us carry out a logging operation. Connect to **vm-internal**, and run the package index update command

`sudo apt-get update`

Enter `exit` to return to your Cloud Shell instance

### Try again to view logs by running the command:
```
gcloud logging read 'resource.type=nat_gateway' \
--limit=10 
--format=json
```

## Task 5. Review
*You created vm-internal, an instance with no external IP address, and connected to it securely using an IAP tunnel. Then you enabled Private Google Access, configured a NAT gateway, and verified that vm-internal can access Google APIs and services and other public IP addresses.*

*VM instances without external IP addresses are isolated from external networks. Using Cloud NAT, these instances can access the internet for updates and patches, and in some cases, for bootstrapping. As a managed service, Cloud NAT provides high availability without user management and intervention.*

*IAP uses your existing project roles and permissions when you connect to VM instances. By default, instance owners are the only users that have the IAP Secured Tunnel User role. If you want to allow other users to access your VMs using IAP tunneling, you need to grant this role to those users.*