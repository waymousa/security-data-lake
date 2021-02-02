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
I had a estimated 17.7 million CloudTrail Events collected over 4 years to process so I created a Cluster using emr-5.31.0, 1 admin node (m5.8xlarge with 20GB EBS volume) and 12 code nodes (m5d.8xlarge).  

Reading the data into a table takes approximately 20-30 seconds when using the schema in the notebook. For the write to ElasticSearch the Spark cluster ran 61079 tasks in 1268.162 seconds so 20 minutes to upload 17.7 million CloudTrail Events is fairly good.  Once the data is uploaded to ElasticSearch you can terminate the EMR cluster in the source account to reduce cost.

You can create the same cluster in each account that is the source for your Cloudtrail Data if you need a multi-account ingestion.

I have also has sucess with creating the notebook first and the letting it create the EMR cluster for you usign autoscaling.  I saw this work when I had 4 years of CloudTrail logs to ingest.  Set the disk on each node to 20GB to be on the safe side.
# 3. Create Elasticearch Cluster
I created an ElasticSearch cluster with 12 data nodes and 3 admin nodes.  I use i3 2XL this ensures plenty of disk space and fast IO to increase indexing throghput.

I use 12 shards for each index and the indexes are created for each month based on the @timestamp for the event.  I have no replicas set.  The template code below sets up the sdl* indexes with this sharding pattern and also sets the max feilds size to 2000.

The ElasticSearch cluster must be created with a public endpoint and not using the VPC.  This is because the EMR cluster will need to be able to access the ElasticSearch endpoint directly in the VPC and thats not possible because you can't set up the EMR cluster in the ElasticSearch security group.  What you could look at is using an EMR VPC endpoint to try to get around this.  

You also need to set up the cluster to use the intenal user database.  The reason for this is because the elasticsearch-hadoop java class only supports http basic authentication so Cognito and IAM won't work.

```bash
PUT _template/sdl_1
{
  "template": "sdl*",
  "settings": {
    "index": {
      "mapping.total_fields.limit": 2000,
      "number_of_shards": 1,
      "number_of_replicas": 0
    }
  }
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

# 7. Scaling, Cost & Deployment
In terms of scaling, the sizing of the EMR and ElasticSearch clkusters is an experiment that you will have to do based on how many events you actually need to process.

Based on the testing I ran, EMR can process and upload 17.7 million Cloud trail events in about 25 minutes using the cluster sizing describes above.  If you have considerably less events to process or you on;y want to process events form a single year then you can adjust eigther the cluster sizing, instance type or filter the events you are ingesting using the /sdl-es/cloudtrailPath parameter.  I had numerous java heap memiory issues witht he notebook when running on smaller instance sizes, in particular the admin node which i have set up on an instance with 64GB of RAM to ensure I didn't have any issues.

The ElasticSearch cluster also needs to be sixzed the the volume of ingestiong you are throwing at it.  Because this data is easy to reload whenever I want it, I opted for no replicas and a number of shards that equalled the number of nodes.  I also set up the nodes on i3 instances to take advantage of the SSD and speed up disk writes.  I sharded across all the nodes because I wanted to make sure that any ingestion to any index would lihgt up the cpu, mewmory and disk on all instances thereby ensuring that the throughput didn;t get strangled with a single node being accessed.  I had numerous java IO issues with the notebook when pushing the events to ElasticSearch so this spread-the-load-wide approach prevented those issues.  

In this use case, where I want to load data for a forensic investingation, I need the ingestion speed so that i can get to the analysis part fast.  I would not recommend leaving clusters of this size running for any length of time as the costs would be large.  The deployment pattern I advise is to have a the notebook stored in a repo somewhere and then pull it and load it into Jupyterlab when you need it.  To deploy the notebook, create a new Jupyter Notebook in the EMR console and select the option to create a cluster for it.  This ensures that the cluster has the right libraries installed.  Next, set up the ElasticSearch cluster.  These do not need to be in the same account, and I would recommend you create the ElasticSearch clusyer in your Security Data lake account.  Last, set the System parameters in the account containig the EMR cluster and make sure that the EMR clustter policy has access to them.  At that point you can run the notebook and ElasticSearch cluster will fill with data and you can start your interactive querying.  You can teminate the EMR cluster at this point.

Once you have completed the forensics, you export the index to an S3 bucket for safekeeping and the shutdown the cluster until you need it next.

I strongly recommend you document this procedure in a runbook and test it out as part of a gameday.  You do not want to be experimenting with the cluster sizes and memory settings when you have a security incident that needs investigating!  