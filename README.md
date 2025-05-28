# AWS Neptune

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

