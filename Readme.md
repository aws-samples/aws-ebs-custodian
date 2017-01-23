Complete EBS Life Cycle Management, from creating and deleting Snapshots.

Disclaimer:

These scripts should be used as guidance for creating a complete EBS Life Cycle Management solution for your production environments. They should not be used in production without modification.

This solution uses Amazon Lambda, EC2, CloudWatch Events, DynamoDB and EC2 Run Command(bonus script). When used together, these functions will :

A. Create Snapshots of your EBS volumes that are attached to your EC2 Instances. This is a accomplish by looking for EC2 Instances that has a certain Tag assigned to them, for example 'snapshot'. Snapshots are then created of the Non Root EBS volumes that are attached to this Instance. Please note that I used the Tag "snapshot" in the snapshot creation script but you can change this what you like.

B. The meta-data for each snapshot is stored in a DynamoDB Table (caled 'Snaps').

C. Update the Status field of the DynamoDB Table with the new status, for example from Pending to Completed

D. Lastly, Your Snapshots will be deleted after the number of days you specified for them to live has expired. The lifetime is stored in the value of the Tag you created for this reason. In my code I added the Tag "Snaplifetime" and a Value of "2" or "4" to my EC2 instances. You are required to do this as well. Your Tag name and values can be what ever you want them to be.

Instructions:

Create Amazon Lambda functions for each files. At the end of this process you should have three Lambda Functions. The IAM role you create for Lambda should allow you access to DynamoDB, EC2 and SNS.

Create Schedule events in Amazon CloudWatch Events. You will need to create three schedules, one that dictates when snapshots are created, one that updates the status of the snapshots in your DynamoDB table and the last one that deletes your snapshots.

Create a Table in DynamoDB to store your Snapshot MetaData. This data facilitates the deletion of your Snapshots and the State status manipulation.

Most Importantly, you are required to Tag your Instances. There are two Tags that you must assign to your Instances. The first Tag tells your Lambda function which Instances are eligible to have their EBS Volumes snapshoted. The other tells the Lambda function how many days these Snapshots should be stored for. In the Snapshot creation function I used the Tag "snapshot" with value of "snapshot" to decide which Instance are eligble to have snapshots created for their Volumes. The second Tag I used is "snapshotlifetime" that holds the value for the number of days this snapshot should be stored for.

I have added the bonus script (reinv-EBS-Snapshot-EC2-RunCmd copy.js) to this repository.This allows you to run shell scripts on your instances before the snapshot is triggered. One thing that this allows you to do is run commands like "fsfreeze -f mountpoint" on Linux Instances which stops writes to the filesystem assigned to your block device. After the snapshot completes you could run the command the opposite command to unfreeze your filesystem which is "fsfreeze -u mountpoint".

If you chose to use the bonus script you will need to install the SSM Agent on each of your EC2 Instances. You will also need to create your Own Document File that EC2 Run Command will use. Please note that the section of the Bonus Script for EC2 Run Command must be modified with your attributes.

AWS Services Used:

Amazon Lambda
Amazon DynamoDB
Amazon Cloud Watch Events
Amazon EC2 Run Command
Amazon EBS
There are four Node.js files in this repository.

1. reinv-ebs-snapshot-creation copy.js :

Used to create a snapshot of an Amazon EBS Volume. This should be used as a base for building your Lmabda functions to achieve Snapshots according Amazon Best Practices.

This Node.js script does an EC2 Describe Instance call and returns results base on a specified filter. The results are in JSON format. The JSON data is parsed and specific information like DeviceName, VolumeId, and Tags are retrieved.

"reinv-ebs-snapshot-creation copy.js" is built to only create Snapshots of Non Root Volumes. Please note that I have built a modifies version of this script that allows you to run commands like "sync" on a Linux server before initiating a Snapshot. EC2 Run Command is used to accomplish this.

Please note that you can leverage EC2 Run Command to take this a step further by flushing the Page Cache on the Linux machine and then un mount the volume Snapshot it and re mount to the instance.

You must schedule this fucntion to run by creating a CloudWatch Event. This schedule can be whatever you want it to. Please note that there are default limits on the number of snapshots you can create, please ensure that your limits meet your needs. You can easily increase your limits by submitting a request to AWS.

2. reinv-snapshot-state-change copy.js

Used to update the "State" field of the Table in DynamoDb with the state of the Snapshot after you have made the CreateSnapshot API call.

You must Create a CloudWatch Event to schedule this function to run multiple times each day. During testing, I schedule this function run every hour. That, however, is overkill

3. reinv-delete-snapshot copy.js :

This script is used to delete each created snapshot after a the period of time you specified. You must add a Tag to your EC2 Instance. The Tag Key can be whatever you want it to be, I used snaplifetime but you can chose to do something else. You must specify a value along with the Tag. The Value is the number of days you would like a Snapshot for this specific Instance to live.

This function calculates the number of days that has passed since your snapshot was created. It then checks the "days" field in the DynamoDB Table against the length of time the snapshots been stored so far. If value in the "days" field is equal to or less than the calculated value, then the snapshot is deleted.

You must schedule this function to run using CloudWatch Events. I generally run it once per day given that I am doing a scan on the DynamoDB table.

BONUS SCRIPT! BONUS SCRIPT! BONUS SCRIPT! BONUS SCRIPT! BONUS SCRIPT!

4. reinv-EBS-Snapshot-EC2-RunCmd copy.js

"reinv-EBS-Snapshot-EC2-RunCmd copy.js" is a modified version of "reinv-ebs-snapshot-creation copy.js ". It is important to note that "reinv-EBS-Snapshot-EC2-RunCmd copy.js" leverages Amazon EC2 Run Command to run shell scripts on the EC2 Instance that the Volumes are being snapshotted for.

The shell script that I ran on my Linux Instances is "sync; echo 3 > /proc/sys/vm/drop_caches" but this could easily have been something far more complex. For example, if I was to write this shell script to achieve a Snapshot according to Amazon best practices then I would simply flush the caches to my block volumes and unmount the volumes then snapshot them and remount when I am finish. You can do this as well as a matter of fact I recommend that you do. Read more on Snapshoting EBS Volumes at "http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-creating-snapshot.html" and EC2 Run Commands at "http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/execute-remote-commands.html".

Amazon EBS Recommendations for creating a Snapshot of an Instane:

" You can take a snapshot of an attached volume that is in use. However, snapshots only capture data that has been written to your Amazon EBS volume at the time the snapshot command is issued. This might exclude any data that has been cached by any applications or the operating system. If you can pause any file writes to the volume long enough to take a snapshot, your snapshot should be complete. However, if you can't pause all file writes to the volume, you should unmount the volume from within the instance, issue the snapshot command, and then remount the volume to ensure a consistent and complete snapshot. You can remount and use your volume while the snapshot status is pending.

To create a snapshot for Amazon EBS volumes that serve as root devices, you should stop the instance before taking the snapshot.

To unmount the volume in Linux, use the following command:

umount -d device_name Where device_name is the device name (for example, /dev/sdh).

After you've created a snapshot, you can tag it to help you manage it later. "

It is important that you :

Add retry logic to these scripts
Amazon Limits are changed/increase to match your needs/requirements
Ensure that the number of writes and reads units that is configured for your DynamoDB table matches the amount of request going to your table
Interesting Reads :

Amazon EC2 Run Command on Linux : http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/execute-remote-commands.html

Amazon Elastic Compute Cloud (Amazon EC2) Run Command lets you remotely and securely manage the configuration of your Amazon EC2 instances, virtual machines (VMs) and servers in hybrid environments, or VMs from other cloud providers. Run Command enables you to automate common administrative tasks and perform ad hoc configuration changes at scale. You can use Run Command from the EC2 console, the AWS Command Line Interface, Windows PowerShell, or the AWS SDKs. Run Command is offered at no additional cost.

Administrators use Run Command to perform the following types of tasks: monitor their systems, install applications on their machines, inventory machines to see which applications are missing or need patching, patch machines, build a deployment pipeline, bootstrap applications, and join instances to a domain, to name a few.

Amazon EBS Snapshots : http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-creating-snapshot.html

After writing data to an EBS volume, you can periodically create a snapshot of the volume to use as a baseline for new volumes or for data backup. If you make periodic snapshots of a volume, the snapshots are incremental so that only the blocks on the device that have changed after your last snapshot are saved in the new snapshot. Even though snapshots are saved incrementally, the snapshot deletion process is designed so that you need to retain only the most recent snapshot in order to restore the volume.

Amazon Lambda : http://docs.aws.amazon.com/lambda/latest/dg/welcome.html

When AWS Lambda executes your Lambda function on your behalf, it takes care of provisioning and managing resources needed to run your Lambda function. When you create a Lambda function, you specify configuration information, such as the amount of memory and maximum execution time that you want to allow for your Lambda function. When a Lambda function is invoked, AWS Lambda launches a container (that is, an execution environment) based on the configuration settings you provided.

Amazon DynamoDB : http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html

Amazon DynamoDB is a fully managed NoSQL database service that provides fast and predictable performance with seamless scalability. DynamoDB lets you offload the administrative burdens of operating and scaling a distributed database, so that you don't have to worry about hardware provisioning, setup and configuration, replication, software patching, or cluster scaling.
