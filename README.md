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
   1. [Step functions](#11)


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

Serverless does not mean there are no servers, but rather that users do not need to manage, provision, or even see them

Serverless in AWS:

- AWS Lambda
- DynamoDB
- AWS Cognito
- AWS API Gateway
- Amazon S3
- AWS SNS & SQS
- AWS Kinesis Data Firehose
- Aurora Serverless
- Step Functions
- Fargate

<a name="11"></a>
## Step functions 

AWS Step Functions is a service that allows us to build serverless visual workflows to orchestrate our Lambda functions. It is considered one of the serverless offerings within AWS.
Key aspects and features of AWS Step Functions include:

- Orchestration of Lambda Functions: Its primary purpose is to help us coordinate and manage multiple AWS Lambda functions, defining the order in which they run and how they interact.
- Workflow Capabilities: Step Functions support various workflow patterns, such as sequencing (running functions one after another), parallel execution, handling conditions (logic branching), managing timeouts, and implementing robust error handling.
- Integrations: Beyond Lambda, Step Functions can integrate with a variety of other services and resources, including EC2, ECS, on-premises servers, API Gateway, SQS queues, ... This broad integration capability allows it to orchestrate complex distributed applications.
- Human Approval: It offers the possibility of implementing human approval features within a workflow, allowing for manual interventions or sign-offs at specific stages.
- Use Cases: Step Functions are suitable for a wide range of applications that involve multiple steps or stages, such as order fulfilment processes, data processing pipelines, web applications, and any general workflow that benefits from visual orchestration.

Think of AWS Step Functions like a digital conductor for an orchestra of cloud services. Just as a conductor directs musicians to play their parts in sequence, in parallel, or based on specific cues, Step Functions directs various AWS services (like Lambda functions) through a predefined workflow, ensuring each step is executed correctly, handling any errors, and allowing for complex logic and integrations. This allows you to build sophisticated, multi-step applications without manually managing the interactions between each component.


<a name="?"></a>
## Lambda 

AWS Lambda is a serverless compute service that allows developers to deploy code as functions without having to manage servers. It was a pioneer in the serverless paradigm, which now encompasses various managed services like databases, messaging, and storage. Serverless does not mean there are no servers, but rather that users do not need to manage, provision, or even see them.
Here's a comprehensive summary of AWS Lambda based on the provided sources:
Core Concepts and Benefits of AWS Lambda
• Virtual Functions, No Server Management: Unlike virtual servers (e.g., Amazon EC2) which are continuously running and require intervention for scaling, Lambda functions are "virtual functions" where there are no servers to manage.
• On-Demand Execution: Lambda functions run on-demand and are limited by time (short executions). This contrasts with EC2 instances which are continuously running.
• Automated Scaling: Scaling in Lambda is automated, removing the need for manual intervention to add or remove servers.
• Easy Pricing: You pay per request and compute time. There's a free tier that includes 1,000,000 AWS Lambda requests and 400,000 GBs of compute time. After the free tier, it costs $0.20 per 1 million requests and $1.00 for 600,000 GB-seconds of compute time. Running AWS Lambda is generally considered "very cheap".
• Resource Allocation: You can easily get more resources per function, with up to 10GB of RAM, and increasing RAM also improves CPU and network performance.
Lambda Capabilities and Integrations
• Language Support: Lambda supports many programming languages, including Node.js (JavaScript), Python, Java (Java 8 compatible), C# (.NET Core), Golang, C# / Powershell, and Ruby. It also offers a Custom Runtime API for community-supported languages like Rust.
• Container Image Support: Lambda can use Container Images, provided the image implements the Lambda Runtime API. For arbitrary Docker images, ECS / Fargate is preferred.
• Extensive AWS Integration: Lambda is integrated with the whole AWS suite of services. Key integrations include CloudWatch Logs, SNS, Cognito, SQS, S3, Kinesis, API Gateway, DynamoDB, CloudFront, CloudWatch Events, and EventBridge.
• Monitoring: Easy monitoring is available through AWS CloudWatch.
• Use Cases:
    ◦ Serverless Thumbnail Creation: A new image in S3 can trigger an AWS Lambda function to create a thumbnail, which is then pushed back to S3, with metadata stored in DynamoDB.
    ◦ Serverless CRON Job: CloudWatch Events (or EventBridge) can trigger an AWS Lambda function to perform a task periodically, for example, every hour.
    ◦ Serverless APIs: API Gateway can expose REST APIs backed by AWS Lambda and DynamoDB, forming a serverless API layer for applications.
    ◦ Workflow Orchestration: AWS Step Functions can orchestrate Lambda functions to build serverless visual workflows, supporting sequencing, parallel execution, conditions, timeouts, and error handling.
Lambda Limits
• Execution Limits:
    ◦ Memory allocation: 128 MB to 10GB (in 1 MB increments).
    ◦ Maximum execution time: 900 seconds (15 minutes).
    ◦ Disk capacity in /tmp: 512 MB to 10GB.
    ◦ Concurrency executions: 1000 (can be increased).
    ◦ Environment variables size: 4 KB.
• Deployment Limits:
    ◦ Compressed .zip deployment size: 50 MB.
    ◦ Uncompressed deployment size (code + dependencies): 250 MB.
    ◦ The /tmp directory can be used to load other files at startup.
Cold Start and SnapStart
• Cold Start (Initialisation Phase): Lambda functions run on-demand, meaning they are not always active. If a function hasn't been recently used, it needs to be initialised before it can execute code, which can introduce a slight delay known as a "cold start" (not explicitly named in source, but described as initialization).
• Lambda SnapStart: This feature significantly improves start-up performance by up to 10x for Java 11 and above functions at no extra cost. When enabled, the function is invoked from a pre-initialised state, avoiding the need for initialisation from scratch. When a new version of a function is published, Lambda performs the initialisation, takes a snapshot of its memory and disk state, and then caches this snapshot for low-latency access. This effectively skips the distinct "Init" phase in the invocation lifecycle.
Lambda in a VPC
• Default Deployment: By default, Lambda functions are launched outside your own VPC (in an AWS-owned VPC), meaning they cannot access resources within your VPC (such as RDS, ElastiCache, internal ELB).
• Accessing VPC Resources: To enable a Lambda function to access resources in your VPC, you must define the VPC ID, subnets, and security groups. Lambda will then create an Elastic Network Interface (ENI) in your specified subnets, allowing it to communicate with your VPC resources like Amazon RDS.
• RDS Proxy Integration: If Lambda functions directly access a database, they might open too many connections under high load. RDS Proxy helps solve this by pooling and sharing DB connections, improving scalability, availability (reducing failover time), and security (enforcing IAM authentication). Lambda functions using RDS Proxy must be deployed in your VPC because RDS Proxy is never publicly accessible.
• Invoking Lambda from RDS & Aurora: Lambda functions can be invoked from within your DB instance (supported for RDS for PostgreSQL and Aurora MySQL) to process data events. This requires allowing outbound traffic from your DB instance to Lambda (e.g., via Public, NAT GW, VPC Endpoints) and setting up necessary IAM permissions.
Lambda@Edge and CloudFront Functions (Edge Computing)
• Edge Functions: These are code snippets you write and attach to CloudFront distributions, running close to your users to minimise latency. They are fully serverless and you only pay for what you use.
• CloudFront Functions:
    ◦ Lightweight JavaScript functions for high-scale, latency-sensitive CDN customisations.
    ◦ Offer sub-millisecond startup times and support millions of requests per second.
    ◦ Used to change Viewer Requests and Responses (after CloudFront receives a request from a viewer, and before CloudFront forwards the response to the viewer).
    ◦ Managed natively within CloudFront.
    ◦ Have limits: <1ms max execution time, 2MB max memory, 10KB total package size. No network or file system access, no access to request body.
    ◦ Use cases: Cache key normalization, header manipulation, URL rewrites/redirects, request authentication/authorization (e.g., JWT validation).
• Lambda@Edge:
    ◦ Lambda functions written in Node.js or Python.
    ◦ Scales to thousands of requests per second.
    ◦ Can change Viewer Request/Response and Origin Request/Response (before forwarding to the origin and after receiving response from origin).
    ◦ Functions are authored in us-east-1 and then replicated globally by CloudFront.
    ◦ Have higher limits: 5-10 seconds max execution time, 128MB up to 10GB max memory, 1MB-50MB total package size. Allows network and file system access, and access to the request body.
    ◦ Use cases: Longer execution times, adjustable CPU/memory, code dependent on third-party libraries (e.g., AWS SDK), network access to external services, file system access, or access to HTTP request body.
AWS Lambda functions are like specialised express delivery drones in a vast logistics network. Instead of having a large, continuously operating warehouse (like a traditional server) always ready, Lambda functions are small, pre-packaged drones. When a specific package needs to be delivered (a request comes in), the appropriate drone is quickly dispatched, performs its single task, and then returns to its charging station, ready for the next order. If it's a very common delivery (high demand), many identical drones can be instantly launched in parallel. And with SnapStart, it's as if the most popular drones are already hovering near the dispatch centre, ready to dart off with their payload at a moment's notice.






**Reference:**

AWS Neptune:

https://www.youtube.com/playlist?list=PLd13Ui6RDb8m5X15ZrQd8XuOXV6RZbPSO 
https://docs.aws.amazon.com/neptune/latest/userguide/feature-overview-db-clusters.html
https://www.youtube.com/watch?v=Y42OwmUF23s&list=PLd13Ui6RDb8m5X15ZrQd8XuOXV6RZbPSO&index=5

