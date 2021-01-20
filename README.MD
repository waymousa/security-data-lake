# User Story
As an AWS Operator I should be able to load all of my CloudTrail archive logs into a single tool so that I can query them for any information I need during a forensic investigation.

# Summary

This project is a tool for importing CloudTrail Events in bulk from S3 sources and storing them in ElasticSearch for easy visual querying.  It does not support streaming so it just lods whatever is in the CloudTrail archive bucket at the time, however with a bit of PySpark knowlege you can easily change the notebook to be a streaming service.

Minimal transformation is done on the CLoudTrail event before it is saved to Elastic Search.  From there its just a matter of setting up your searches and doing adhoc querying of the data.

This took me a long time to get working and a fair number of false starts and dead ends.  I learned a lot about PySpark in the process!

I have sucessfully tested this with a CloudTrail bucket containing 0.5 million events zipped in the usual CloudTrail format.  Cluster sizing reflects this, although further tuning can be done for larger deployments.  Using spot pricing makes this solution cheap enough to have running all the time for streaming applications should you wish to modify the notebook to support it.

# Steps

# 1. Set up Service policy
aws iam create-service-linked-role --aws-service-name es.amazonaws.com
# 2. Create EMR Cluster
I had a estimated 0.5 million ElasticSeaqrch Events to process so I created a Clusyer using emr-5.31.0, 1 admin node and 6 code nodes.  To keep it simple, just use a public one to get yourself started.
# 3. Create Elasticearch Cluster
I created an E;asticSearch cluster with 6 data nodes and 3 admin nodes.  This ensurs you have plenty of disk space to play with while you index.
# 4. Run Jupyter notebook in Jupyterlab
Start the Jupyter Notebook in Jupyterlab.  The notebook is contained in the root of this project.  You will need to amend the following Sections:
<<<<<<< HEAD
cloudTrailLo
=======
cloudTrailLo
>>>>>>> 0dbe6548748614ba6f21f628aad9a69e95134761
