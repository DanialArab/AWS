Amazon Web Services

1. [AWS Neptune](#1)
   1. [How to set up a Neptune Cluster](#2)



<a name="1"></a>
# AWS Neptune

Amazon's fully managed graph database service. AWS Neptune abstracts away the underlying infrastructure, allowing us to focus on data modeling, ingestion, and query logic—without the overhead of managing servers, scaling, or patching. 

<a name="2"></a>
## How to set up a Neptune Cluster
We can create a Neptune Cluster by
- using an AWS CloudFormation template or
- manually using the AWS Management Console

If we use a **<a href="https://docs.aws.amazon.com/neptune/latest/userguide/get-started-create-cluster.html">CloudFormation template</a>**, it creates all the required resources for us, without having to do everything by hand.

The required steps to create a Neptune Cluster using the Console can be found <a href="https://docs.aws.amazon.com/neptune/latest/userguide/manage-console-launch-console.html">here</a> . Here is the summary of the settings we would like to change, while leaving others as default:

Cluster type: Provisioned or Serverless

Templates: Production vs. Development and testing 

DB Cluster name 

Create storage configurations: Standard vs. I/O-Optimized 

Instance type: Memory optimized vs. Burstable classes

Availability and Durability: whether or not we want to create read replica in different availability zone (AZ)

Network and security: we need to specify the VPC in which the Cluster will be created (we do have a specific VPC named ml-vpc, you can find its ID in your AWS console in MachineLearning-Dev). Note that after a database is created, we can't change the VPC selection. We also need to select the subnet groups associated with our VPC and also the VPC security groups.

Tags

Notebook configuration: this will bug out and you will get an error, so don’t worry about this while creating your Cluster, we will take care of this later in this document. 

A side note useful for production: We can also create Neptune Global DB spanning across multiple regions enabling low-latency global reads and providing fast recovery in the rare case where an outage affects an entire AWS Region.


- CSV is the only data format supported by Gremlin
- 

## First, I need to create a Neptune DB cluster. Some notes:

- Why tags:
I don't strictly need tags on my Amazon Neptune database, but they are highly recommended for the following reasons:

1. Cost Allocation

Tags help you track AWS costs per project, team, environment, or feature.

Example: If multiple teams use different Neptune clusters, tagging helps break down the bill:

    Environment=dev, Team=data-science, Project=entity-resolution.

2. Resource Organization

Easily filter and find resources in the AWS Console or CLI using tags.

Makes your infrastructure more manageable at scale.

3. Access Control (via IAM)

You can use tag-based conditions in IAM policies to restrict who can modify or delete specific Neptune databases.

Example: Only users with Project=analytics tag access can touch databases tagged that way.

4. Automation

Tags can trigger lifecycle automation (e.g., backups, shutdowns, alerts).

Example: Auto-delete dev resources older than 30 days if tagged Environment=dev.

5. Compliance and Auditing

Some orgs enforce tagging to meet internal governance or external compliance standards.

📝 Example Tags for a Neptune Cluster

        Key	Value
        
        Name	entity-resolution-db

## Second I need to set up a security group for the Neptune DB

I need an appropriate security group with a name of something like NeptuneAccess with an inbound rule allowing TCP traffic on port 8182. A quick reminder: Outbound traffic is all allowed by default in the security group. 

## Notebooks in Neptune

I do need an appropriate IAM role which allows me call different services SageMaker, Bedrock, and S3. It seems that Notebooks in Neptune will be re-directed to SageMaker and from SageMaker to our it'll be re-directed to the actual Jupyter Notebook. 

## Load data into the Neptune DB

I need to create an Endpoint as a middle-man connecting Neptune DB within a VPC and S3. So I need to create one, I think Gateway endpoint is the best compared to the Interface endpoint since it is free for S3 and DynamoDB.

Reference: 

https://www.youtube.com/playlist?list=PLd13Ui6RDb8m5X15ZrQd8XuOXV6RZbPSO 

https://docs.aws.amazon.com/neptune/latest/userguide/feature-overview-db-clusters.html

https://www.youtube.com/watch?v=Y42OwmUF23s&list=PLd13Ui6RDb8m5X15ZrQd8XuOXV6RZbPSO&index=5

