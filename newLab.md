**Activity 01: Creating a static website**  

Downoad this [zip](https://github.com/nvaws/planetup/archive/refs/heads/main.zip) file  
Create a new bucket in any region of your choice  
Click on the bucket name  
Click on upload  
Drag and Drop the index.html and assets folder, click on upload  
Go to Permission, modify the Block public access and uncheck the box confirm/save  
Update bucket policy to make your website publiclly readable  
Bucket Policy: update the bucket name in the below policy and paste  
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AddPerm",
            "Effect": "Allow",
            "Principal": "*",
            "Action": [
                "s3:GetObject"
            ],
            "Resource": [
                "arn:aws:s3:::<your bucket name>/*"
            ]
        }
    ]
}
```
Go to properties and enable website hosting feature  
See if you can visit the website url in browser  

**Activity 02: Serving the website via CloudFront distribution**  

Let us serve this website using CloudFront now  
Go to CloudFront service
Click on "Create distibution"  
- Origin domain: selet your s3 bucket from the dropdown  
-  Ignore the recommandation to "Use websie endpoint" if you see a message  
- Select "Legacy access identities"  
- Click on Create new OAI  
- Select "Yes, update the bucket policy"  
- leave everything else as default  
- Default root object - optional: index.html  

Click on Create distribution  

The distribution url will be created and will go into deployment mode. You can copy the distribution url and visit in few minutes after the distribution is deployed  

Check if there is Bucket policy created in your S3 bucket, try to understand what does it mean.  

Your website is now being served by the CloudFront url only and is not directly accessible using the previous "Bucket website endpoint"


**Activity 03: Creating a DynamoDB Table to store prospect's data**  

Create a dynamodb table  
Table name: prospects  
Primary key: name  
Create  
copy the endpoint: arn:aws:dynamodb:us-east-1:123456789123:table/prospects  

create an item with fields name, email, message, phone

**Activity 04: Creating a Lambda function to store prospect's data into DynamoDB table**  

create a lambda function
Function name: AnyName
Runtime: Node.js 16.x
Permissions: Assign a role that has S3 and DynamoDB full access.
Create Function
Edit code inline - delete the existing code in index.js and paste the one here. Update region name (line 5, 13) and table name (line 27) in the code. Save.
Keep the tab open

**Activity 05: Creating a REST API using AWS API Gateway to trigger te Lambda function** 

go to APIGateway  
Get Started  
Create API  
Choose the protocol: REST  
Create new API: New API  
Settings  
API Name: NewProspectsAPI  
Endpoint Type: Regional  
Create API  

You API will get created and you should be in resources section. Click on Actions dropdown and Create Resource 
Resource Name: prospects
Enable API Gateway CORS: yes
Create Resource

Click again on Actions dropdown and Create Methode
Select POST and click on the check box to save
on the right side of the screen
Integration type: Lambda Function
Use Lambda Proxy integration: yes
Lambda Region: Select your region from dropdown
Lambda Function: type the name of your Lambda function
Save
It will ask your consent for adding permission to Lambda Function, say OK

Click on Action Dropdown - Deploy API
Deployment stage: New Stage
Stage name: dev 
Deploy
Go to the Stages section, expand the dev stage and click on POST
Copy the Invoke URL
if you go to Lambda page now, you will find that a trigger has been added.

finally update the Invoke URL in the javascript file (line 9) inside assets -> custom folder, and upload it again 

Now you can go to the planetup site webpage, refresh it once, put some data in the form and submit.
if everything went fine then you shund see an entry in your dynamo DB table.

remove the public permission policy from s3 bucket
turn on the "Block public access (bucket settings)"


Cleanup steps
disable and delete the CloudFront distribution (takes time)
DynamoDB Table
API in API Gateway
Lambda Function 
S3 bucket



Lab Completed.
