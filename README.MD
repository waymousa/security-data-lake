# User Story
As an AWS Operator I should be able to load all of my CloudTrail archive logs into a single tool so that I can query them for any information I need during a forensic investigation.

# Summary

This project is a tool for importing CloudTrail Events in bulk from S3 sources and storing them in ElasticSearch for easy visual querying.  It does not support streaming so it just loads whatever is in the CloudTrail archive bucket at the time, however with a bit of PySpark knowlege you can easily change the notebook to be a streaming service.

Minimal transformation is done on the CLoudTrail event before it is saved to Elastic Search.  From there its just a matter of setting up your searches and doing adhoc querying of the data.

This took me a long time to get working and a fair number of false starts and dead ends.  I learned a lot about PySpark in the process!

I have sucessfully tested this with a CloudTrail bucket containing 0.5 million events zipped in the usual CloudTrail format.  Cluster sizing reflects this, although further tuning can be done for larger deployments.  Using spot pricing makes this solution cheap enough to have running all the time for streaming applications should you wish to modify the notebook to support it.

# Steps

# 1. Set up Service policy
```bash
aws iam create-service-linked-role --aws-service-name es.amazonaws.com
```
# 2. Create EMR Cluster
I had a estimated 1.7 million CloudTrail Events to process so I created a Cluster using emr-5.31.0, 1 admin node and 6 code nodes.  Reading the data into a table takes approximately 20-30 seconds when using the schema in the notebook.  
You can create the same cluster in each account that is the source for your Cloudtrail Data if you need a multi-account ingestion.

I have also has sucess with creating the notebook first and the letting it create the EMR cluster for you usign autoscaling.  I saw this work when I had 4 years of CloudTrail logs to ingest.  Set the disk on each node to 20GB to be on the safe side.
# 3. Create Elasticearch Cluster
I created an ElasticSearch cluster with 6 data nodes and 3 admin nodes.  This ensures you have plenty of disk space to play with while you index.  1.7 million CloudTrail events used up 5.5 GB of disk in a single index and takes about 12 minutes to index using the elasticsearch-hadoop maven package. 
The ElasticSearch clusyter must be created with a public endpoint and not using the VPC.  This is because the EMR cluster will need to be able to access the ElasticSearch endpoint directly in the VPC and thats not possible because you can't set up the EMR cluster in the ElasticSearch security group.  What you could look at is using an EMR VPC endpoint to try to get aroudn this.  
You also need to set up the cluster to use the intenal user database.  The reason for this is because the elasticsearch-hadoop java class only supports http basic authentication so Cognito and IAM won't work.
You might find that you get the odd index that will cause a failure because the maximum fileds limit of 1000 is reached.  Set this to 2000 like so:
```bash
PUT sdl*/_settings
{
  "index.mapping.total_fields.limit": 2000
}
```
You can use the same ElasticSearch cluster for all the accounts you want to load data from.
# 4. Parameter Store
The parameters for accessing the ElasticSearch clusyer from EMR are put in parameter store.  Parameters you need to define for the notebook are listed here:
|SM Parameter Name|Type|Description|Example|
|---|---|---|---|
|/sdl-es/cloudtrailPath|String|s3 url for the CloudTrail log bucket|s3n://BUCKET_NAME_HERE/AWSLogs/\*/CloudTrail/\*/\*/\*/\*/|
|/sdl-es/esEndpoint|String|The ElasticSearch Endpoint without the protocol in front of it|ELASTICSEARCH_URI|
|/sdl-es/esIndex|String|The index name|sdl-\{\@timestamp\|YYYY\.MM\}/cloudtrail|
|/sdl-es/master/password|SecureString|ElasticSearch user password|YOUR_ELASTICSEARCH_USER_PASSWORD|
|/sdl-es/master/user|String|ElasticSearch user name|YOUR_ELASTICSEARCH_USER_NAME|
# 5. Update the EMR_EC2_Default role
You need to allow the EMR nodes to access parameter store in order to run the job.  Create an inline policy and post the following json into it.  Amend the account number.
```bash
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": [
                "ssm:GetParametersByPath",
                "ssm:GetParameter"
            ],
            "Resource": "arn:aws:ssm:us-east-1:your-account-number:parameter/sdl-es*"
        },
        {
            "Sid": "VisualEditor2",
            "Effect": "Allow",
            "Action": "ssm:DescribeParameters",
            "Resource": "*"
        }
    ]
}
```
# 6. Run Jupyter notebook in Jupyterlab
Start the Jupyter Notebook in Jupyterlab.  The notebook is contained in the root of this project.  The Notebook imports the elasticsearch-hadoop from maven repo and deploys the boto3 libraries to allow access to AWS Systems manager parameter store.  Code and notes on code are in the notebook itself.

You should see the indexes being created in ElasticSearch and filling up.  The indexes are create for each month using the sdl-YYYY.MM/cloudtrail pattern.  

500,000 documents seems to be about 1.5GB with a default 5 primary and 1 replica shard.