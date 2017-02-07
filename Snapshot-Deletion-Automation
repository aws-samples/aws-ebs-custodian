//This Lambda Function checks to see if the number of days thi Snapshot has been stored and deletes it if it has reached its expiry date.
//I have set a schedule in CloudWatch Events to  execute this function once per day. 

var AWS = require('aws-sdk'); 
var util = require('util');
AWS.config.update({ region: 'us-west-2' }); //ADD REGION HERE 

//Variable Declaration for AWS API Libraries
var ec2 = new AWS.EC2({ apiVersion: 'latest' });
var dynoClient = new AWS.DynamoDB.DocumentClient({ region: 'us-west-2' }); //IF YOUR REGION ISNT US-WEST-2, THEN PLEASE FEEL FREE TO CHANGE IT

exports.handler = (event, context, callback) => {

var params = {
    TableName: 'Snaps', //REPLACE 'SNAPS' WITH A DYNAMODB TABLE YOU CREATED 
};


dynoClient.scan(params, function(err, data) {
    if (err) console.log(err); // an error occurred
    else {

            data.Items.forEach(function(snapitem) {
                
                
                var dayz = snapitem.Date;

                var today = Date.now();

                var daydiff = today - dayz;

                var numofdays = daydiff/86400000;

                var numD = Math.round(numofdays);
                

            var snapId = snapitem.SnapshotId; //snapId holds the DynamoDb table "Snaps" primary key or HashId
           
            var snaplifetime = snapitem.day; 

              
            if (snaplifetime =< numD) { 
           
        // if ((snaplifetime === numD || snaplifetime < numD)) { 
                
                var delparams = {

                    SnapshotId: snapId

                };

                ec2.deleteSnapshot(delparams, function(err, data) {
                  
                  if (err) {
                        console.log(err, err.stack);
                    } 
                    
                    else {
                        //console.log("Snapshot  :   " +data.SnapshotId+ "    deleted successfully");

                        var rmvparams = {
                            TableName: 'Snaps',
                            Key: {

                                "SnapshotId": snapId

                            },
                            ConditionExpression: "#estate =:s",
                                                    
                            ExpressionAttributeNames: {
                                "#estate": "day"
                            },

                            ExpressionAttributeValues: {
                                ":s": snaplifetime
                            }

                        };
                        
                        dynoClient.delete(rmvparams, function(err, data) {
                                                        
                            if (err) {
                                console.error("Unable to delete item. Error JSON:", JSON.stringify(err, null, 2));
                                //callback(err, null);
                                
                                
                            } 
                            
                            else {
                          
                            console.error("Delete item:  " +snapId);
                            //callback(null, data);
                                                            
                            } 
                            
                        });
                        
                          
                        
                        
                    }
                    
                });

                

            } 

            else { console.log("Nothing to delete"); }

        }); 


    } 

});

};
