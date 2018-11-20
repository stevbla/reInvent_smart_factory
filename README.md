# MFG303 - Enable your Smart Factory with the AWS Industrial IoT Reference Solution

## Overview
This workshop will walk you through connecting industrial devices (PLCs Data) in your simulated factory environment to the AWS Cloud.

The workshop will walk you through the following steps:
1. Setup Greengrass as a Gateway
2. Deploy a Lambda Function with Node Red to act as the data concentrator.
3. Create and Setup the PLC Simulator
4. Configure Node Red to talk between the PLC and Greengrass

## Pre Requisites.
To complete this workshop you will need an AWS Account with a Key Pair setup. You will also need the Microsoft RDP Client installed on your PC.

To ensure the workshop is successful we suggest that you use the **US-EAST-1** region for all AWS Services and tasks.


## Setting up the Environment.

To setup the environment for this workshop you will need to download the following [CloudFormation Template](cfntemplates/mfg303-cfn.yaml).

1. Log into the AWS Console and select **CloudFormation** from the Services menu.
2. Click **Create Stack**
3. Choose to ***Upload a template to Amazon S3*** and click Next.
4. Name the Stack: **mfg303**
5. Enter a Username and Password for C9User and C9Passwd respectivily, these will be the credentials you use to log into the Cloud9 interface.
6. Select the EC2 Key Pair you would like to use.
7. The default Instance Type of t2.micro can be left as default.
8. Click Next & Next
9. Tick the "I acknowledge that AWS CloudFormation might create IAM resources." box and click **Create**

The required resources will now be created for you, once completed successfully you can move onto the next step.

> To help you in future steps of the workshop I suggest you take note (copy to a text file) of the value of **C9IdeURL** in the output of CloudFormation.

## Setting up the PLC Simulator.

Once the CloudFormation template successfully deploys you can carry on with the below steps.

1. From the AWS Console menu select EC2 and click on "Running Instances".
2. Click on the instance with the Name **PLC Windows Host** and copy the Public DNS name for the instance.
3. Open the Microsoft Remote Desktop Client and add a new Desktop connect. Copy in the Public DNS name of the PLC EC2 instance into the PC Name dialog box.
4. Back in the AWS Console click on **Connnect** for the EC2 instance.
5. Click **Get Password**
6. Copy and Paste or upload the public key pair of the key you selected in the CloudFormation and click **Decrypt**
7. Double Click the EC2 instance in the RDP client and enter **Administrator** as the User Name and copy the password from the previous step into the **Password** box. If prompted click **Continue** for the certificate validation pop-up box.
8. When the RDP Window launches, click Yes on the "Network Discovery" window if prompted.

### Installing the PLC Simulator Software.

1. Launch Chrome on the Windows PLC machine and navigate to the following URL https://bedrockautomation.com/download
2. Complete the form on the page and click "DOWNLOAD"
3. Click on "BEDROCK IDE V1.8" and then "DOWNLOAD"

The software is approximately 1.4G, so it will take a couple of minutes to download.

4. Once downloaded, using Windows Explorer browse to the Downloads folder and right click on the **Bedrock-IDE-1.8.6772.46976.697f0bec-Setup** and select Extract.
5. Double click on **Bedrock-IDE-1.8.6772.46976.697f0bec-Setup.exe** and click "Run" on the security warning pop-up.
6. Click Install on the InstallShield Wizard window.

It will take a couple of minutes to install pre-requisite software.

7. When the Bedrock IDE V1.8 window appears, click "Next".
8. Accept the license agreement, click "Next".
9. Accept all the default options and keep selecting "Next" till the installation starts.

The installation will take around 10 minutes, so we will move onto configuring Greengrass.

## Configuring Greengrass

### Creating a Greengrass Group.

Make sure you are in the US-EAST-1 region.

1. From the AWS Console select the **IoT Core** service.

If you have never used IoT core before, click “Get started”.

2. On the left hand menu select **Greengrass**.
3. Within the main window under "Define a Greengrass Group" click "Get Started".
4. Click "Use easy creation"
5. Name your GG Group **PLCGateways**, click Next
6. Accept the default name for the Core and Click Next.
7. Click "Create Group and Core"
8. Click on “Download the resources as tar.gz” and store the downloaded files anywhere on your local hard disk.

> Typically, it is required to download the Greengrass release you want to use for the chip architecture of the device you are using. In this case, we can skip this step as we have already installed Greengrass on the EC2 instance via the CloudFormation template.

9. Click “Finish” at the bottom of the screen.

Within IAM create a new Role for Greengrass called **Greengrass_ServiceRole** which has the following policies attached AmazonS3FullAccess, CloudWatchLogsFullAccess, AWSGreengrassResourceAccessRolePolicy.

### Configuring the Greengrass Device.

1. From your Web Browser open a new tab or Window and browse to the Cloud9 URL which is referenced in the Outputs from the CloudFormation template **C9IdeUrl**.
2. Depending on your Web Browser accept/acknowledge the security warning around the website.
3. When you get the credential dialog box, enter the Username and Password you used when deploying the CloudFormation template.
4. From the Cloud9 IDE, click **File** > **Upload Local Files...** and select the tar.gz file that you downloaded from the Greengrass setup step.
5. From the Cloud9 IDE terminal window (bottom of the page), run the below command to extract the files.

```bash
sudo tar -xzvf ~/*-setup.tar.gz -C /greengrass
```
6. Next lets Start Greengrass with the below command.

```bash
sudo /greengrass/ggc/core/greengrassd start
```

### Create a new Lambda Function.

1. From the Cloud9 Terminal

```bash
tar -xvf nodeRedLambda.tar -C nodeRedLambda
cd nodeRedLambda
```
Edit package.json and add the following lines

```json
"node-red-contrib-modbus": "^4.1.1",
"node-red-contrib-opcua": "^0.2.32",
```

```bash
npm install
zip -r nodeRedLambda.zip ./*
```

Right click on the file and select "Download"

From the AWS Console create a new Lambda function from scratch called **nodeRedFunction** with a Node.js 6.10 runtime. Choose "Create a new role from one or more templates" with a role name of nodeRedRole.

In the Lambda function window under **Function Code** choose to Upload the Function via a ZIP file and hit save at the top of the page.

From the action menu select "Publish new version", type First version and then choose Publish. Select Create alias with a name of **GG_nodeRedFunction** with a version of 1.

From the IoT Services menu select the PLCGateways group under the Greengrass > Groups menu. Then select Add Lambda from the Lambdas menu. Choose "Use existing Lambda" and select nodeRed and then select the Alias you created followed by Finished.

Once created click on **...** on the Lambda funciton and select Edit configuration.

Set the Memory limit to 250M and for the Lambda Lifecycle choose long-lived.

Make sure you click update.

Go back to the Greengrass group menu and choose Subscriptions
Click Add Subscription, setting the Source as the nodeRed Lambda function and the Target as the IoT Cloud.
Click next and set **#** as the Topic Filter.
Once done click Finish.

From the Resources side menu, click "Add a Local Resource".
Name the resource nodeRed with type volume. Source path = /userdir and Destination path = /nodered. Set Automatically add OS group permissions of the Linux group that owns the resource,  select the nodeRed Lambda function with Read and Write and then click Save.

ml resource: source: locate or upload a model in S3, select the uploaded file from s3, local path: /flows
 - upload S3/myflow.json.zip to the bucket.

Click the Settings menu and click on Add Role for Group Role and select the Greengrass_ServiceRole followed by Save.

Finally under the Action menu for the Group, click Deploy using Automatic Detection and Grant Permissions.

> If the account previously has not been used with AWS Greengrass, the deployment can potentially fail because AWS Greengrass cannot assume the service role required to execute the deployment.

Via the Cloud9 terminal and ensuring your awscli credentials are configured, run the below commands.

```bash
gg_service_arn=$(aws iam get-role --role-name Greengrass_ServiceRole --query Role.Arn)
[ec2-user@ip-192-168-128-85 ~]$ aws greengrass associate-service-role-to-account --role-arn ${gg_service_arn//\"/}
```
Once associated re-run the Deploy.

## PLC PC

From the AWS Console go to EC2.
Click on "Launch Instance" and search for the following AMI within the community.

MFG303 Bedrock Automation - ami-07a329552c30a599f

Click Next and Select t2.xlarge as the instance type following by "Configure Instance Details"
Select the VPC and Subnet for the instance which was created as part of the CloudFormation template and where the GG EC2 instance is residing.

Click Next on the Storage and add a Name Tag called "PLC Windows."

On the Security Group, select the security group which has a description of "Windows PLC SG"

Click Review and Launch and then Launch.

Once Launched, take note of the Public DNS IP and using RDP connect to it.

The Username and Password to connect to the instance is provide on screen.

## Open Bedrock IDE.

Double click on Bedrock IDE icon on the desktop.

Click File > Open and select the file DataStreams.






### Bedrock Configuration.

When the install WIzard prompts you, select "I have read the information" and click Next.
Click Finish.








 ```javascript
 var fs = require('fs');
 var request = require('request');
 var http = require('http');
 var express = require("express");
 var RED = require("node-red");

 var RED_IP = 'http://127.0.0.1';
 var RED_PORT = 1880;

 var GG_ML_RESOURCE = '/flows'; //local path of the affiliated Greengrass Machine Learning Resource, which is a compressed file in S3
 var NEWFLOW_FILE = GG_ML_RESOURCE + '/myflow.json'; //name of the flow file in GG_ML_RESOURCE
 var GG_LOCAL_RESOURCE = '/nodered';
 var BACKUP_FOLDER = GG_LOCAL_RESOURCE + '/backup';
 var USER_DIR = GG_LOCAL_RESOURCE + '/userdir';

 // Create an Express app
 var app = express();
 // Add a simple route for static content served from 'public'
 app.use("/",express.static("public"));

 // Create a server
 var server = http.createServer(app);

 // Create the settings object
 var settings = {
     httpAdminRoot: "/red",
     //disableEditor: true,        //Disable the UI in production
     httpNodeRoot: "/api",
     userDir: USER_DIR,
     functionGlobalContext: {
       require:require
     }    //enables global context

     //TODO: authentication
 };


 function backupExistingFlows(fileName) {
   var options = {
     //https://nodered.org/docs/api/admin/methods/get/flows/
     uri: RED_IP + ':' + RED_PORT + '/red/flows',
     method: 'GET'
   };

   request(options, function(error, response, body) {
     if (!error && response.statusCode == 200) {
       fs.writeFile(fileName, body, function(err) {
         if (error) {
           return console.log('Flow file writing error: ' + error);
         }
         console.log('backed up the existing flows');

         console.log('loading the new flows after backup');
         loadNewFlows();
       });

     } else {
       console.log('HTTP Error while reading existing flow:', error);
     }
   });

 }

 function loadNewFlows() {
   console.log('load the latest flows available');
   fs.readFile(NEWFLOW_FILE, 'utf8', function(error, data) {
     if (error) {
       return console.log('Flow file reading error: ' + error);
     }

     var options = {
       //https://nodered.org/docs/api/admin/methods/post/flows/
       uri: RED_IP + ':' + RED_PORT + '/red/flows',
       method: 'POST',
       json: JSON.parse(data)
     };

     console.log('file was read, requesting REST API');
     request(options, function(error, response, body) {
       if (!error && response.statusCode == 204) {
         console.log('loaded the new flows');
       } else {
         console.log('HTTP Error while loading new flow:', error);
       }
     });
   });
 }



 function init() {
   console.log('Node-RED initializing...');
   RED.init(server,settings);

   app.use(settings.httpAdminRoot,RED.httpAdmin);  // Serve the editor UI from /red
   app.use(settings.httpNodeRoot,RED.httpNode);  // Serve the http nodes UI from /api

   server.listen(RED_PORT);

   return RED.start().then(() => {
     console.log('Node-RED server started.');

     //WARNING: In order to use REST API, Node-RED must be running. Undeserved flow may run before loading!
     //This can be solved by copying the flow to userDir as a JSON file. Must be validated!

     var fileName = BACKUP_FOLDER + '/flow-' + Date.now() + '.json';
     console.log('backing up the existing flows to: ' + fileName);
     backupExistingFlows(fileName);

   })
 };


 init();


 exports.handler = function handler(event, context, callback) {
   console.log('This handler does not update the flow on GG core. Don\'t forget to deploy!');
   callback(undefined, { result: 'This handler does not update the flow on GG core.' });
 };
 ```
