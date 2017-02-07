/* BONUS FUNCTION!!!!!!! 

This Node.js Lambda Function does the following :

1. Retrieves the Meta Data for all Instances with the Tag 'snapshot' 

2. Uses EC2 Run Command to run a shell script on the EC2 instance to flush the Page Cache to the Block Devices attached to it.
   Amazon Recommends that when initiating a Snapshot, all writes must be stopped to the voleume and the volume unmount snapshot and remounted. 
   Please note that you can accomplish this by creating your own EC2 Run Command Document with the Linux Commands that can accomplish.
   My Uses of EC2 Run Command in this script is to show that it is possible to execute a shell command on the instance to perform some function     
   that adds value to the task you are trying to accomplish. 
 
3. Creates a Snapshot for all Non Root Volume
4. Store the Meta Data for that Snapshot in the DynamoDB Table 'Snaps' [Please change Snaps to your DynamoDB Table Name
5. Reads the Tag SnapLifeTime tha stores the number of days this Snapshot must be stored for 

PLEASE REMEMBER THAT THIS IS A BASE FOR CREATING A WORK FLOW MANAGING YOUR SNAPSHOTS. IT DOES NOT CONTAIN CERTAIN BEST PRACTICES LIKE UN MOUNTING BEFORE SNAPSHOTING A VOLUME. YOU CAN HOWEVER MODIFY THIS SCRIPT TO ACCOMPLISH THIS.

AWS Services Used: 

a. EC2: Instances exist only within EC2 today
b. Lambda : Runs This Function
c. DynamoDB: Stores Meta Data for Snapshots that are created
d. CloudWatch Events : Schedules and Triggers this script
E. EC2 RUN COMMAND


*/


var AWS = require('aws-sdk');
var util = require('util');

AWS.config.update({region: 'us-west-2'});
var ssm = new AWS.SSM();

exports.handler = (event, context, callback) => {

//Variable Declaration for AWS API Libraries
var ec2 = new AWS.EC2({apiVersion: 'latest'});
const dynDB = new AWS.DynamoDB.DocumentClient({region: 'us-west-2'});


var string;
var num = 0;
var idcount = 0;
var ins;
var Id = [];
var icount = 0;
var op = 0;
var bp =[];
var tnum= -1;
var ace;
var snaps = [];
var vol;
var vparams;
var b=0;
var tagz;
var tag = [];
var uniqueArray = [];
var mrkr = 0;

//Defines a parameter that contains a filter for a specific Tag. In this case the value of the Tag is 'snapshot'. 
//It can, however, be what ever you want it to be 
var param = { Filters: [ { Name: 'tag-value', Values: ['snapshot'] } ] };
				
				
var request = ec2.describeInstances(param);

//array to hold object variables for param 
var snapshots = [];
var i = [];
var a = [];
var groot = [];
var glob = [];


request.on('success', function(response) {
   

 
	for( var item in response.data.Reservations) {  	
		var instances = response.data.Reservations[item].Instances;
		
		
		for ( var instance in instances) {
				
				//"group" is a variable that stores the attributes of the Instances that are returned
				var group = instances[instance];
				
				
				//"rootdev" is used to grab the root device information. 
				//"groot" is an array that is used to store each root device information
				var rootdev = instances[instance].RootDeviceName;
				groot.push(rootdev);
				
			
				//Commence parsing the attributes of the instance
				var dat = JSON.stringify(group, 2);
				var runner = JSON.parse(dat, (key, value)=>{
										
					
					//Grabs Instance-Id. This is used as a parameter in the EC2 Run Command Function
					if (key ==='InstanceId')
					{
						ins = value.toString();
						
					} //End if : 1
					
					//Grabs the DeviceName of the Volumes attaced to the EC2 Instances. Used in a for loop to check if its a Root Device. 
					//Root Devices are not Snapshot in this Lambda F(n).
					if (key ==='DeviceName')
					{
						ace = value.toString();
						i.push(ace);
					
					}//End if : 2
					
					if(key === 'VolumeId')
					{
						vol = value.toString();
						a.push(vol);
						icount++;
								
					} //End if : 3
					
					if(key === 'Key')
					{
						tagz = value.toString();
						
						
					}//End if : 4
					 
					
					if((key === 'Value') && (tagz === 'snaplifetime'))
					{
						
						b = Number(value);
						
						for (sol = 0; sol < icount; ++sol)
						{
						
								tag.push(b);
								Id.push(ins);
								
						}
						
						icount = 0;	
									
					} //Closes if((key === 'Value') && (tagz === 'snaplifetime'))
									
					
				}); //End of PARSING
				
		
		} //Closed the "for ( var instance in instances) {"
			
		
	} // Closes the "for( var item in response.data.Reservations) { "
	
	
	//Removing Duplicate Items from the Array "groot" 
	uniqueArray = groot.filter(function(elem, pos) {
		return groot.indexOf(elem) == pos;
	});
	
	//bp is an array that stores the root device name. This fucntion combines all the elements in the array into one string. 
	bp.push(uniqueArray.toString()) ;
	bp.join();
	
	//Stores the combined elements of the array in one single variable
	string = bp[0].toString();
	
	
	// Looping through array  "i" and filtering out Root Devices
	for (numsnaps = 0; numsnaps < i.length; ++numsnaps) {
		
		
			//Checks to see if the volume device name is a root device name. 
			if (string.includes(i[numsnaps].toString())){  mrkr = 0; }
				
			
			else { ++mrkr;  }
				
		//--------------------- Checks to see if the volume in play is the root volume, if not then a snapshot is initiated  ------------
			
		if (Number(mrkr) >= 1) {
			
			
			var vict = a[numsnaps].toString(); //
			var success = Id[numsnaps];
			
			
			
			var params = {
				VolumeId: vict,			
				DryRun: false
				};
		
			/*--EC2 RUN COMMAND COMMENCES--------------------------------------------------------------------------------------------------

			//PLEASE REPLACE SOME CONDITION WITH YOUR CONDITIONS OR SIMPLY REMOVE THIS if STATEMENT IF:
			ALL YOUR INSTANCES ARE LINUX AND THEY HAVE THE SSM AGENT INSTALLED ON THEM */

			if (success ==='SOME CONDITION'){  	
				
						
			//------------------------  Amazon EC2 Run Command : Flush Page Cache on Linux Instances to Block Devices -------------
			
			var runparams = {
				DocumentName: 'DOCUMENT-NAME', /* required */
				InstanceIds: [ /* required */
					success
					/* more items */
				],
				
				/* Please note that I have chosen to do a PageCache Flush but you can chose to stop writes and unmounts
				the block devices as recommended by AWS. */
				Comment: 'Flushes Linux PageCache to Block Devices', 
				
				/* References an EC2 Document that has the command "sync; echo 3 > /proc/sys/vm/drop_caches" defined. */
				DocumentHash: 'ADD Document HASH Here', 
				DocumentHashType: 'Sha256',
				NotificationConfig: {
					NotificationArn: 'arn:aws:sns:AWS-REGION:ACCOUNTNUMBER:NOTIFICATION_NAME',
					NotificationEvents: [
						'All'
						/* more items */
					],
					NotificationType: 'Command'
				},
				OutputS3BucketName: 'YOUR-S3_BUCKET-NAME', //Outputs generated from your shell script will be stored here
				OutputS3KeyPrefix: 'PREFIX-OF-FOLDER-IN-S3-Bucket',
				ServiceRoleArn: 'arn:aws:iam::ACCOUNTNUMBER:role/ROLE-NAME',
				TimeoutSeconds: 3600
			};
			
			//API CALL TO THE SSM AGENT RUNNING ON YOUR LINUX INSTANCES
			ssm.sendCommand(runparams, function(err, data) { 
				if (err) console.log(err, err.stack); // an error occurred
				else     console.log(data);           // successful response
			});
			
			//-----------------------------------------------------------------------------------------------------------------------
			
			
			} //CLOSES THE " if (success === 'some condition') { "
			
			
			//Uses the VolumeId stored in the "vict" variable to create snapshots
			ec2.createSnapshot(params, function(err, data) {
						
			if (err) console.log(err, err.stack); // an error occurred
			else { // successful response
					
				tnum = tnum+1;
				op = Number(tag[tnum]);	
				
				 				
				 vparams = {
					
					Item: {
					
					SnapshotId: data.SnapshotId,
					VolumeId: data.VolumeId,
					State: data.State,
					StartTime: data.StartTime,
					OwnerId: data.OwnerId,
					VolumeSize: data.VolumeSize,
					Tags: data.Tags,
					Encrypted: data.Encrypted,
					Date: Date.now(),
					day: op
					
					
					},
					
					/*Snaps is the name a DynamoDB Table that I created. You can however create your own table and call it whatever you want 
					like Ninja! , No, seriously, you can call it Ninja :) */

					
					TableName: 'Snaps' 					
										
					}; //Closes "vparams"
					
					//Writes Snapshot Attirbutes to a DynamoDB Table "Snaps"
					dynDB.put(vparams, function(err, data) 
					
					{
						if (err) console.log(err, err.stack); // an error occurred
						else { console.log('Successful Write!'); }

											
					}); //Closes DynamoDB PUT Call
				
					

				} // Closes the ELSE
				
			

				}); //Closes EC2.CreateSnapshot Function
				

			
				} //Closes if (Number(mrkr) >= 1) Function
		
	
		
				} // Closes for (numsnaps = 0; numsnaps < i.length; ++numsnaps) Function 
		
			
			//console.dir();
				

			
  }). // ON.Request Function (Very First Function)


  on('error', function(response) {
    console.log("Error!");
  }).

send();

};
