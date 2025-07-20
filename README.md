# Amazon Web Services

1. [AWS Neptune](#1)
   1. [How to set up a Neptune Cluster](#2)
   2. [How to interact with our Neptune Cluster](#3)
   3. [Internet access using Internet Gateway, NAT Gateway, or Endpoints](#4)
   4. [How to load data](#5)
      1. [Bulk loading](#6)
      2. [Small loads using Query Languages](#7)
   5. [Neptune Workbench Notebook](#8)
   6. [AWS Neptune Analytics](#9)
2. [Serverless](#10)
   1. [Step function](#11)


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

- Cluster type: Provisioned or Serverless
- Templates: Production vs. Development and testing
- DB Cluster name
- Create storage configurations: Standard vs. I/O-Optimized
- Instance type: Memory optimized vs. Burstable classes
- Availability and Durability: whether or not we want to create read replica in different availability zone (AZ)
- Network and security: we need to specify the VPC in which the Cluster will be created. Note that after a database is created, we can't change the VPC selection. We also need to select the subnet groups associated with our VPC and also the VPC security groups.
- Tags
- Notebook configuration: this will bug out and you will get an error, so we don’t need to worry about this while creating your Cluster, we will take care of this later in this document. 

A side note useful for production: We can also create <a href="https://docs.aws.amazon.com/neptune/latest/userguide/neptune-global-database.html">Neptune Global DB spanning across multiple regions</a> enabling low-latency global reads and providing fast recovery in the rare case where an outage affects an entire AWS Region.

A side note on why we need tags:

QWe don't strictly need tags on my Amazon Neptune database, but they are highly recommended for the following reasons:

- Cost Allocation:

Tags help you track AWS costs per project, team, environment, or feature.

Example: If multiple teams use different Neptune clusters, tagging helps break down the bill:

    Environment=dev, Team=data-science, Project=entity-resolution.

- Resource Organization:

Easily filter and find resources in the AWS Console or CLI using tags.

Makes your infrastructure more manageable at scale.

- Access Control (via IAM):

We can use tag-based conditions in IAM policies to restrict who can modify or delete specific Neptune databases.

Example: Only users with Project=analytics tag access can touch databases tagged that way.

- Automation:

Tags can trigger lifecycle automation (e.g., backups, shutdowns, alerts).

Example: Auto-delete dev resources older than 30 days if tagged Environment=dev.

- Compliance and Auditing:

Some orgs enforce tagging to meet internal governance or external compliance standards.

Example Tags for a Neptune Cluster

        Key	Value
        
        Name	my-db

<a name="3"></a>
## How to interact with our Neptune Cluster

After creating our cluster, we do need to interact with it to:

- load in the data
- perform analytics
- query the database 

Preferably, our S3 bucket, where our data to be loaded into Neptune Cluster live, should be in the **same region** as our Neptune Cluster. If the bucket is in a different region, we need cross-region access that always go over the public internet. Also VPC endpoints cannot be created for a bucket sitting in a different region. 

Amazon S3 is a global service and exists outside our VPC. The access to S3 objects is strictly controlled through bucket policies and IAM permissions. So in order to access S3 objects we need:

- appropriate permissions — the IAM role attached to Neptune (or its associated service) must have permissions to access the specific S3 bucket or object, and
- internet access 

<a name="4"></a>
## Internet access using Internet Gateway, NAT Gateway, or Endpoints

As mentioned, our Neptune cluster lives inside a VPC, which by default does not have internet access: therefore, it cannot access resources like Amazon S3. So we do need to explicitly configure internet access by using an Internet Gateway (IGW), NAT Gateway, or a VPC Endpoint (Gateway is a preferred endpoint vs interface endpoints for S3 and DynamoDB).

<a name="5"></a>
## How to load data 

There are several approaches to ingest data into Neptune, each suited for different use cases, data volumes, and source formats. Based on our projects' stage we need to choose the most efficient and appropriate method. Broadly these methods are categorized as:

- Bulk loading
- Small loads using Query Languages 

<a name="6"></a>
### Bulk loading 

This is the primary and most efficient method recommended for loading large volumes of data into Neptune. It involves placing our data files in an S3 bucket and then initiating a bulk load job. The bulk process involves sending a POST request to the Neptune loader endpoint using 

- curl,
- AWS CLI, or
- AWS SDK (boto3)

<a name="7"></a>
### Small loads using Query Languages 

For smaller datasets, or when we need to make incremental updates or add individual elements, we can use Neptune's supported query languages. In this method, we generally do not need to stage data in S3 first. This is a key differentiator from the bulk loading method. 

Using query languages directly interacts with the Neptune database endpoint, which allows us to insert, update, or delete individual graph elements (vertices/nodes and edges for property graphs) directly through API calls or client tools. This method is best suited for:

- smaller datasets
- incremental updates, like adding new data points as they arrive, or making small modifications to existing data
- real-time inserts: like our application needs to add data to the graph immediately in response to user actions or events.
- interactive development and testing: manually adding data during development or testing phases

Neptune supports two primary query languages for graph data manipulation: **Gremlin** and **SPARQL**.

<a name="8"></a>
## Neptune Workbench Notebook

<a href="https://docs.aws.amazon.com/neptune/latest/userguide/graph-notebooks.html#graph-notebooks-workbench">Workbench Notebook</a> is an IDE managed by SageMaker, which is tailored specifically for Neptune, offering a convenient environment to load data, visualize, analyze, and run Gremlin or SPARQL queries. It is a suggested method for both bulk and small loads because:

- for small, interactive, or incremental loads, we can use the direct query language magics like %%gremlin, %%sparql, %%oc
- for large loads, we can use boto3 to call Neptune Bulk Loader API from within a notebook cell

We can load data into Neptune without using the Workbench Notebook by utilizing boto3 and initiating a bulk loader job directly. However, the Workbench Notebook offers several advantages: it provides graph and query visualizations, like the one in the following figure, comes pre-configured with libraries such as Gremlin, SPARQL, boto3, and requires zero setup. Also, it comes with sample applications, tutorials, and the code snippets. Also, graph-explorer is <a href="https://docs.aws.amazon.com/neptune/latest/userguide/visualization-graph-explorer.html">automatically deployed </a> with the Workbench Notebook. These features can make it a bit easier to get started. Here is the link to their <a href="https://github.com/aws/graph-notebook">GitHub page</a>.

![](https://github.com/DanialArab/images/blob/main/AWS/workbench_nb_visualization.png)

This <a href="https://aws.amazon.com/awstv/watch/bf4e7142537/">link </a> from AWS could be useful to get started with the Workbench and Neptune (the recorded screen quality is not perfect, though).

<a name="9"></a>
## AWS Neptune Analytics

AWS Neptune Analytics a graph DB engine which stores large datasets in memory providing a quick way to analyze graph data. It supports:

- libraries of optimized graph analytic algorithms,
- low-latency graph queries, and
- vector search capabilities within graph traversals.

<a name="10"></a>
# Serverless 

<a name="11"></a>
## Step function 

AWS Step Functions is a service that allows us to build serverless visual workflows to orchestrate our Lambda functions. It is considered one of the serverless offerings within AWS.
Key aspects and features of AWS Step Functions include:
• Orchestration of Lambda Functions: Its primary purpose is to help us coordinate and manage multiple AWS Lambda functions, defining the order in which they run and how they interact.
• Workflow Capabilities: Step Functions support various workflow patterns, such as sequencing (running functions one after another), parallel execution, handling conditions (logic branching), managing timeouts, and implementing robust error handling.
• Integrations: Beyond Lambda, Step Functions can integrate with a variety of other services and resources, including EC2, ECS, on-premises servers, API Gateway, SQS queues, ... This broad integration capability allows it to orchestrate complex distributed applications.
• Human Approval: It offers the possibility of implementing human approval features within a workflow, allowing for manual interventions or sign-offs at specific stages.
• Use Cases: Step Functions are suitable for a wide range of applications that involve multiple steps or stages, such as order fulfilment processes, data processing pipelines, web applications, and any general workflow that benefits from visual orchestration.

Think of AWS Step Functions like a digital conductor for an orchestra of cloud services. Just as a conductor directs musicians to play their parts in sequence, in parallel, or based on specific cues, Step Functions directs various AWS services (like Lambda functions) through a predefined workflow, ensuring each step is executed correctly, handling any errors, and allowing for complex logic and integrations. This allows you to build sophisticated, multi-step applications without manually managing the interactions between each component.


<a name="?"></a>
## Lambda 








**Reference:**

AWS Neptune:

https://www.youtube.com/playlist?list=PLd13Ui6RDb8m5X15ZrQd8XuOXV6RZbPSO 
https://docs.aws.amazon.com/neptune/latest/userguide/feature-overview-db-clusters.html
https://www.youtube.com/watch?v=Y42OwmUF23s&list=PLd13Ui6RDb8m5X15ZrQd8XuOXV6RZbPSO&index=5

