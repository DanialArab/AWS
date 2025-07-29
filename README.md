# Amazon Web Services 

In this repo, I document my understanding of AWS after getting the following certificates:
- **<a href="https://www.credly.com/badges/75349987-1f49-41cf-a155-34f9fd018499/linked_in?t=sv9uqx">AWS Solution Architect Associate</a>** Date obtained: March 24, 2025 Expires: March 24, 2028
- **<a href="https://www.credly.com/badges/f2678b53-c145-4c53-8c8a-6c4f4ce98368">AWS Certified Cloud Practitioner</a>** Date obtained: March 24, 2022 Expires: March 24, 2028
- AWS ML Specialty (working on this Certificate, scheduled to take the exam on Oct. 5, 2025)

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
   1. [Lambda](#11)
      1. [Core Concepts and Benefits of AWS Lambda](#12)
      2. [Lambda Capabilities and Integrations](#13)
      3. [Use Cases](#14)
      4. [Lambda Limits](#15)
      5. [Cold Start and SnapStart](#16)
      6. [Lambda in a VPC](#17)
      7. [Customization At The Edge](#18)
         1. [CloudFront Functions](#19)
         2. [Lambda@Edge](#20)
   2. [Step functions](#21)
   3. [Amazon DymanoDB](#22)
      1. [Core Concepts and Structure](#23)
      2. [Read/Write Capacity Modes](#24)
      3. [DynamoDB Accelerator (DAX)](#25)
      4. [DynamoDB Streams](#26)
      5. [DynamoDB Global Tables](#27)
      6. [Time To Live (TTL)](#28)
      7. [DynamoDB - Backups for Disaster Recovery](#29)
      8. [Integration with Amazon S3](#30)
      9. [Serverless Architecture Context](#31)
    4. [AWS API Gateway](#32)
       1. [Core Functionality](#33)
       2. [Integrations](#34)
       3. [API Gateway Endpoint Types](#35)
       4. [Security](#36)
    5. [Amazon Cognito](#37)
       1. [Cognito User Pools](#38)
          1. [Cognito User Pools Integration](#39)
       3. [Cognito Identity Pools (Federated Identity)](#40)
       4. [Cognito vs. IAM](#41)
    6. [Serverless Architectures](#42)
       1. [Mobile Application: MyTodoList](#43)
       2. [Serverless Hosted Website: MyBlog.com](#44)
       3. [Micro Services Architecture](#45)
       4. [Software Updates Offloading (Optimising Existing EC2 Application)](#46)
3. [Containers on Cloud](#47)
   1. [Docker Containers Management on AWS](#48)
   2. [ECS](#49)
   3. [EKS](#50)


  


   
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
## Lambda 

AWS Lambda is a serverless compute service that allows developers to deploy code as functions without having to manage servers. It was a pioneer in the serverless paradigm, which now encompasses various managed services like databases, messaging, and storage.

Here's a comprehensive summary of AWS Lambda:

<a name="12"></a>
### Core Concepts and Benefits of AWS Lambda

- **Virtual Functions, No Server Management**: Unlike virtual servers (e.g., Amazon EC2) which are continuously running and require intervention for scaling, Lambda functions are "virtual functions" where there are no servers to manage.
- **On-Demand Execution**: Lambda functions run on-demand and are limited by time (short executions). This contrasts with EC2 instances which are continuously running.
- **Automated Scaling**: Scaling in Lambda is automated, removing the need for manual intervention to add or remove servers.
- **Easy Pricing**: You pay per request and compute time. There's a free tier that includes 1,000,000 AWS Lambda requests and 400,000 GBs of compute time. After the free tier, it costs $0.20 per 1 million requests and $1.00 for 600,000 GB-seconds of compute time. Running AWS Lambda is generally considered **very cheap**.
- **Resource Allocation**: You can easily get more resources per function, with up to 10GB of RAM, and **increasing RAM also improves CPU and network performance.**

<a name="13"></a>
### Lambda Capabilities and Integrations

- **Language Support**: Lambda supports many programming languages, including Node.js (JavaScript), Python, Java (Java 8 compatible), C# (.NET Core), Golang, C# / Powershell, and Ruby. It also offers a Custom Runtime API for community-supported languages like Rust.
- **Container Image Support**: Lambda can use Container Images, provided the image implements the Lambda Runtime API. For arbitrary Docker images, ECS / Fargate is preferred.
- **Extensive AWS Integration**: Lambda is integrated with the whole AWS suite of services. Key integrations include CloudWatch Logs, SNS, Cognito, SQS, S3, Kinesis, API Gateway, DynamoDB, CloudFront, CloudWatch Events, and EventBridge.
- **Monitoring**: Easy monitoring is available through AWS CloudWatch.

<a name="14"></a>
### Use Cases:

- Serverless Thumbnail Creation: A new image in S3 can trigger an AWS Lambda function to create a thumbnail, which is then pushed back to S3, with metadata stored in DynamoDB.
- Serverless CRON Job: CloudWatch Events (or EventBridge) can trigger an AWS Lambda function to perform a task periodically, for example, every hour.
- Serverless APIs: API Gateway can expose REST APIs backed by AWS Lambda and DynamoDB, forming a serverless API layer for applications.
- Workflow Orchestration: AWS Step Functions can orchestrate Lambda functions to build serverless visual workflows, supporting sequencing, parallel execution, conditions, timeouts, and error handling.
 
<a name="15"></a>
### Lambda Limits

- Execution Limits:
   - Memory allocation: 128 MB to 10GB (in 1 MB increments).
   - Maximum execution time: 900 seconds (15 minutes).
   - Disk capacity in /tmp: 512 MB to 10GB.
   - Concurrency executions: 1000 (can be increased).
   - Environment variables size: 4 KB.

- Deployment Limits:
   - Compressed .zip deployment size: 50 MB.
   - Uncompressed deployment size (code + dependencies): 250 MB.
   - The /tmp directory can be used to load other files at startup.

<a name="16"></a>
### Cold Start and SnapStart

- **Cold Start (Initialisation Phase)**: Lambda functions run on-demand, meaning they are not always active. If a function hasn't been recently used, it needs to be initialised before it can execute code, which can introduce a slight delay known as a "cold start" (not explicitly named in source, but described as initialization).
- **Lambda SnapStart**: This feature significantly improves start-up performance by up to 10x for Java 11 and above functions at no extra cost. When enabled, the function is invoked from a pre-initialised state, avoiding the need for initialisation from scratch. When a new version of a function is published, Lambda performs the initialisation, takes a snapshot of its memory and disk state, and then caches this snapshot for low-latency access. This effectively skips the distinct "Init" phase in the invocation lifecycle.

<a name="17"></a>
### Lambda in a VPC

- **Default Deployment**: By default, Lambda functions are launched outside our own VPC (in an AWS-owned VPC), meaning they cannot access resources within our VPC (such as RDS, ElastiCache, internal ELB).
- **Accessing VPC Resources**: To enable a Lambda function to access resources in our VPC, we must define the VPC ID, subnets, and security groups. Lambda will then create an Elastic Network Interface (ENI) in our specified subnets, allowing it to communicate with our VPC resources like Amazon RDS.
- **RDS Proxy Integration:** If Lambda functions directly access a database, they might open too many connections under high load. RDS Proxy helps solve this by pooling and sharing DB connections, improving scalability, availability (reducing failover time), and security (enforcing IAM authentication). Lambda functions using RDS Proxy must be deployed in our VPC because RDS Proxy is never publicly accessible.
- **Invoking Lambda from RDS & Aurora**: Lambda functions can be invoked from within our DB instance (supported for RDS for PostgreSQL and Aurora MySQL) to process data events. This requires allowing outbound traffic from our DB instance to Lambda (e.g., via Public, NAT GW, VPC Endpoints) and setting up necessary IAM permissions. DB instance must have the required permissions to invoke the Lambda function (Lambda Resource-based Policy & IAM Policy)

<a name="18"></a>
### Customization At The Edge

Edge Functions: These are code snippets you write and attach to CloudFront distributions, running close to your users to minimise latency. They are fully serverless and you only pay for what you use. CloudFront provides two types of Edge functions: 
- CloudFront Functions &
- Lambda@Edge

Use case: Customize the CDN content

<a name="19"></a>
#### CloudFront Functions:

- Lightweight JavaScript functions for high-scale, latency-sensitive CDN customisations.
- Offer sub-millisecond startup times and support millions of requests per second.
- Used to change Viewer Requests and Responses (Viewer Request: after CloudFront receives a request from a viewer, and Viewer Response: before CloudFront forwards the response to the viewer).
- Managed natively within CloudFront.
- Have limits: <1ms max execution time, 2MB max memory, 10KB total package size. No network or file system access, no access to request body.
- Use cases: Cache key normalization, header manipulation, URL rewrites/redirects, request authentication/authorization (e.g., JWT validation). 

<a name="20"></a>
#### Lambda@Edge:
- Lambda functions written in Node.js or Python.
- Scales to thousands of requests per second.
- Can change Viewer Request/Response and Origin Request/Response.
- Functions are authored in one AWS region like us-east-1 and then replicated globally by CloudFront.
- Have higher limits: 5-10 seconds max execution time, 128MB up to 10GB max memory, 1MB-50MB total package size. Allows network and file system access, and access to the request body.
- Use cases: Longer execution times, adjustable CPU/memory, code dependent on third-party libraries (e.g., AWS SDK), network access to external services, file system access, or access to HTTP request body.

AWS Lambda functions are like specialised express delivery drones in a vast logistics network. Instead of having a large, continuously operating warehouse (like a traditional server) always ready, Lambda functions are small, pre-packaged drones. When a specific package needs to be delivered (a request comes in), the appropriate drone is quickly dispatched, performs its single task, and then returns to its charging station, ready for the next order. If it's a very common delivery (high demand), many identical drones can be instantly launched in parallel. And with SnapStart, it's as if the most popular drones are already hovering near the dispatch centre, ready to dart off with their payload at a moment's notice.


<a name="21"></a>
## Step functions 

AWS Step Functions is a service that allows us to build serverless visual workflows to orchestrate our Lambda functions. It is considered one of the serverless offerings within AWS.
Key aspects and features of AWS Step Functions include:

- Orchestration of Lambda Functions: Its primary purpose is to help us coordinate and manage multiple AWS Lambda functions, defining the order in which they run and how they interact.
- Workflow Capabilities: Step Functions support various workflow patterns, such as sequencing (running functions one after another), parallel execution, handling conditions (logic branching), managing timeouts, and implementing robust error handling.
- Integrations: Beyond Lambda, Step Functions can integrate with a variety of other services and resources, including EC2, ECS, on-premises servers, API Gateway, SQS queues, ... This broad integration capability allows it to orchestrate complex distributed applications.
- Human Approval: It offers the possibility of implementing human approval features within a workflow, allowing for manual interventions or sign-offs at specific stages.
- Use Cases: Step Functions are suitable for a wide range of applications that involve multiple steps or stages, such as order fulfilment processes, data processing pipelines, web applications, and any general workflow that benefits from visual orchestration.

Think of AWS Step Functions like a digital conductor for an orchestra of cloud services. Just as a conductor directs musicians to play their parts in sequence, in parallel, or based on specific cues, Step Functions directs various AWS services (like Lambda functions) through a predefined workflow, ensuring each step is executed correctly, handling any errors, and allowing for complex logic and integrations. This allows you to build sophisticated, multi-step applications without manually managing the interactions between each component.

<a name="22"></a>
## Amazon DymanoDB

Amazon DynamoDB is a fully managed, highly available NoSQL database service that offers transaction support and replication across multiple Availability Zones (AZs). It is designed to scale to massive workloads, capable of handling millions of requests per second, trillions of rows, and hundreds of terabytes of storage, all while maintaining fast and consistent performance with single-digit millisecond latency. It integrates with IAM for security, authorisation, and administration, offers low cost, and provides auto-scaling capabilities. DynamoDB requires no maintenance or patching and is always available, offering both Standard and Infrequent Access (IA) Table Classes.

<a name="23"></a>
### Core Concepts and Structure

- DynamoDB is comprised of Tables.
- Each table requires a Primary Key determined at creation time.
- Tables can contain an infinite number of items (equivalent to rows).
- Each item has attributes (and they're pretty much columns) that can be added over time and can be null.
- The maximum size of an item is 400KB (so DynamoDB is not there to store very large objects).
- It supports rapidly evolving schemas.
- Supported data types include Scalar Types (String, Number, Binary, Boolean, Null), Document Types (List, Map), and Set Types (String Set, Number Set, Binary Set). An example table might include User_ID, Game_ID, Score, and Result attributes, with User_ID potentially serving as a Partition Key and Game_ID as a Sort Key.


<a name="24"></a>
### Read/Write Capacity Modes

DynamoDB offers two modes to manage read/write throughput:

- Provisioned Mode (default):
    - You specify the number of reads/writes per second,
    - requiring capacity planning beforehand.
    - You pay for provisioned Read Capacity Units (RCU) and Write Capacity Units (WCU),
    - with auto-scaling options available for RCU & WCU.
- On-Demand Mode:
    - Reads and writes automatically scale up/down with your workloads,
    - eliminating the need for capacity planning.
    - This mode is more expensive as you pay for what you use and is ideal for unpredictable workloads or sudden spikes.

<a name="25"></a>
### DynamoDB Accelerator (DAX)

- DAX is a fully-managed, highly available, seamless **in-memory cache** for DynamoDB.
- Its primary purpose is to solve **read congestion by caching data**, providing microseconds latency for cached information.
- It's compatible with existing DynamoDB APIs, meaning it doesn't require application logic modification.
- The default TTL (Time To Live) for cache entries is 5 minutes.
- While DAX caches individual objects, queries, and scans, Amazon ElastiCache can be used for storing aggregation results.
    
<a name="26"></a>
### DynamoDB Streams

- DynamoDB Streams provide an ordered stream of item-level modifications (create, update, delete) to a table.
- Use cases include
    - reacting to changes in real-time (e.g., sending a welcome email to new users),
    - real-time usage analytics,
    - inserting into derivative tables,
    - implementing cross-region replication, and
    - invoke AWS Lambda on changes to your DynamoDB table

-  DynamoDB Streams have
    - a 24-hour retention period, and
    - a limited number of consumers.
    - They can be processed using AWS Lambda Triggers or the DynamoDB Stream Kinesis adapter.
      
- Kinesis Data Streams (newer) offers
    - longer retention (1 year) and
    - supports a higher number of consumers.
    - It can be processed by AWS Lambda, Kinesis Data Analytics, Kinesis Data Firehose, or AWS Glue Streaming ETL.
  
Changes from a DynamoDB table can flow through DynamoDB Streams to Kinesis Data Streams for further processing for purposes like SNS messaging, analytics in Amazon Redshift, archiving to Amazon S3, or indexing in Amazon OpenSearch.

<a name="27"></a>
### DynamoDB Global Tables

- This feature allows a DynamoDB table to be accessible with low latency in multiple AWS regions.
- It enables Active-Active replication, meaning applications can read and write to the table in any configured region.
- Enabling DynamoDB Streams is a prerequisite for Global Tables.

  
<a name="28"></a>
### Time To Live (TTL)

- DynamoDB's TTL feature allows for automatic deletion of items after an expiry timestamp.
- Common use cases include reducing stored data by keeping only current items, adhering to regulatory obligations, and managing web sessions.

<a name="29"></a>
### DynamoDB - Backups for Disaster Recovery

- **Continuous backups** are supported via point-in-time recovery (PITR), which can be optionally enabled for the last 35 days. PITR allows recovery to any specific time within the backup window, creating a new table upon recovery.
- **On-demand backups** provide full backups for long-term retention until explicitly deleted. These backups do not affect performance or latency and can be managed through AWS Backup, which also enables cross-region copy. Similar to PITR, the recovery process creates a new table.

<a name="30"></a>
### Integration with Amazon S3

- **Export to S3**: Requires PITR enabled and can export data from any point in the last 35 days. This process does not affect the read capacity of your table and is useful for data analysis, retaining snapshots for auditing, and ETL processes before re-importing. Exports can be in DynamoDB JSON or ION format.
- **Import from S3**: Supports importing CSV, DynamoDB JSON, or ION formats. This process does not consume any write capacity, creates a new table, and logs import errors in CloudWatch Logs.

<a name="31"></a>
### Serverless Architecture Context
- DynamoDB is a key component in serverless architectures.
- It can be directly integrated with AWS Lambda (e.g., for CRUD operations through an API Gateway).
- For mobile applications, DynamoDB serves as the database layer, often combined with DAX for high read throughput and API Gateway for caching.
- In serverless hosted websites, DynamoDB (potentially with Global Tables) provides the database backend for dynamic content, and DynamoDB Streams can trigger Lambda functions for actions like sending welcome emails to new users.
- In microservices architectures, DynamoDB can be a chosen database for individual services.

Think of DynamoDB as a vast, highly organised library where every book (item) has a unique address (primary key), and you can quickly find any book you need, even if the library spans multiple buildings across different cities (Global Tables). You can also hire a fast librarian (DAX) to keep the most popular books on hand for immediate access (caching). If someone borrows, returns, or adds a book, a special log (DynamoDB Stream) records every detail, allowing other services to react in real-time. The library even automatically discards books past their expiry date (TTL) and has a sophisticated system for full or continuous backups, ensuring no book is ever truly lost.

<a name="32"></a>
## AWS API Gateway

AWS API Gateway is a service that allows you to create, publish, maintain, monitor, and secure APIs at any scale. It acts as a "front door" for applications to access data, business logic, or functionality from your backend services.

<a name="33"></a>
### Core Functionality

When used with AWS Lambda, API Gateway means no infrastructure to manage, providing a serverless API solution. It supports the WebSocket Protocol and can handle API versioning (e.g., v1, v2) and different environments (development, test, production). It also provides features for transforming and validating requests and responses, generating SDKs and API specifications, and caching API responses.

<a name="34"></a>
### Integrations

API Gateway can integrate with various backend services:
- **Lambda Function**: It offers an easy way to expose REST APIs backed by AWS Lambda functions, allowing API Gateway to invoke Lambda functions.
- **HTTP Endpoints**: It helps to expose HTTP endpoints in the backend. It can expose internal HTTP APIs (e.g., on-premise systems, Application Load Balancers), enabling you to add features like rate limiting, caching, user authentication, and API keys to existing HTTP backends.
- **AWS Services**: You can expose any AWS API through API Gateway, such as starting an AWS Step Function workflow or posting a message to SQS. This allows for adding authentication, public deployment, and rate control for AWS services. An example illustrates its use with Kinesis Data Streams and Kinesis Data Firehose to send records and store JSON files in Amazon S3.
    
<a name="35"></a>
### API Gateway Endpoint Types

API Gateway offers three main endpoint types:

- **Edge-Optimized (default)**: Ideal for global clients, routing requests through CloudFront Edge locations to improve latency. The API Gateway itself still resides in a single region.
- **Regional**: Best for clients within the same region. You can manually combine this with CloudFront for more control over caching strategies and distribution.
- **Private**: Only accessible from your Virtual Private Cloud (VPC) using an interface VPC endpoint (ENI). Access is defined using a resource policy.

<a name="36"></a>
### Security

API Gateway provides robust security features for user authentication and HTTPS:

- User Authentication:
    - IAM Roles: Useful for internal applications.
    - Cognito: Provides identity for external users, such as mobile users.
    - Custom Authorizer: Allows you to implement your own custom logic for authorization.
- HTTPS Security: It integrates with AWS Certificate Manager (ACM) for custom domain names and HTTPS. If using an Edge-Optimized endpoint, the certificate must be in us-east-1. For Regional endpoints, the certificate must be in the API Gateway region. You must also set up a CNAME or A-alias record in Route 53.

In essence, AWS API Gateway acts like a digital traffic controller for your applications. Just as a traffic controller directs cars (requests) from various roads (clients) to their correct destinations (backend services) efficiently and securely, API Gateway manages the flow of API calls, ensuring they reach the right services (Lambda, HTTP endpoints, AWS services) while applying rules for security, performance, and scaling

<a name="37"></a>
## Amazon Cognito

Amazon Cognito is a service designed to give users an identity to interact with your web or mobile application. It allows you to manage user identities and provide them with secure access to your applications and AWS resources.
Cognito comprises two main components:

<a name="38"></a>
### Cognito User Pools

   - It provides sign-in functionality for application users.
   - It enables you to create a serverless database of users for your web and mobile apps.
   - Key user features include simple login with username/email and password, password reset, email and phone number verification, and multi-factor authentication (MFA).
   - User Pools also support federated identities, allowing users to sign in using third-party providers like Facebook, Google, or SAML.
   - It **integrates with API Gateway and Application Load Balancer**. When used with an Application Load Balancer, it can authenticate users and evaluate Cognito tokens. Similarly, with API Gateway, it can be used to authenticate and retrieve tokens.

<a name="39"></a>
#### Cognito User Pools Integration

When a user authenticates, User Pools gives back a token, and that token can be checked by services like API Gateway or Application Load Balancer. That's how they authenticate the user and allow them to use our backend services. It's a pretty seamless flow from login to secure backend access. 

![](https://github.com/DanialArab/images/blob/main/AWS/CUP_Integrations.png)

<a name="40"></a>
### Cognito Identity Pools (Federated Identity)

- Identity Pools provide AWS (temporary) credentials to users so they can access AWS resources directly. This is crucial for giving users temporary AWS credentials.
- The source of these users can include Cognito User Pools, 3rd party logins (social identity providers), and more.
- Once users obtain these credentials, they can access AWS services directly (such as private S3 buckets or DynamoDB tables) or through API Gateway.
- The IAM policies applied to these temporary credentials are defined within Cognito Identity Pools. These policies can be customised for fine-grained control, potentially even based on the user_id. Default IAM roles can be set for both authenticated and guest users.
- A common use case involves a web or mobile application logging in and obtaining a token from a social identity provider or Cognito User Pool, then exchanging this token for temporary AWS credentials validated by Cognito Identity Pools, allowing direct access to AWS resources.

<a name="41"></a>
### Cognito vs. IAM

In essence, Cognito helps you manage user authentication and authorisation, distinguishing itself from IAM which is generally for **internal AWS users or services**, whereas Cognito is tailored for "hundreds of users" or "mobile users" who "authenticate with SAML".

You can think of Amazon Cognito as a digital doorman for your application. The User Pool is like the main entrance, checking credentials (username/password, social logins) and verifying identity (MFA) before allowing users into your application's lobby. Once inside, the Identity Pool acts like a special pass desk inside the lobby, where users can exchange their application entrance ticket for a temporary, tailored badge (AWS credentials) that grants them direct, limited access to specific, secure rooms (AWS resources like S3 buckets or databases) within the larger AWS building.

<a name="42"></a>
## Serverless Architectures

Let's explore several serverless and cloud-optimised architectural patterns, designed to address various application requirements such as scalability, high availability, cost efficiency, and performance.

<a name="43"></a>
### Mobile Application: MyTodoList

This architecture is designed for a mobile application requiring
- exposing a REST API with HTTPS,
- a serverless backend,
- direct user interaction with S3 folders,
- managed serverless authentication,
- users can write and read to-dos, but they mostly read them
- a scalable database with high read throughput.

- Core Components:
     - Amazon API Gateway: Exposes the application as a REST API over HTTPS.
     - AWS Lambda: Handles the backend logic invoked by the API Gateway.
     - Amazon DynamoDB: Serves as the database, designed for high read throughput and scalability.
     - Amazon Cognito: Provides managed serverless authentication for users.
     - Amazon S3: Stores user-specific data, allowing users to directly interact with their own folders.
     - AWS STS (Security Token Service): Used by Cognito to generate temporary credentials, enabling users to access AWS resources like S3 directly with restricted policies. This pattern can also be applied to DynamoDB and Lambda.

- Key Optimisations:
     - Caching: Utilises DAX (DynamoDB Accelerator) for caching reads on DynamoDB to handle high read throughput. 
     - Security: Authentication and authorisation are managed by Cognito and STS.
     - Caching the responses at the Amazon API Gateway level. This is also something we can do And this is also a very good one if you think that the answers never really change, and that you can start caching a few responses for some few API routes.

![](https://github.com/DanialArab/images/blob/main/AWS/mobile_app.png)

<a name="44"></a>
### Serverless Hosted Website: MyBlog.com

This architecture focuses on a globally scalable website where content (blogs) is often read but rarely written. Some of the website is purely static files, the rest is a dynamic REST API. So it combines static files with a dynamic REST API, incorporates caching, and includes flows for new user welcome emails and photo thumbnail generation.

- Core Components:
   - Amazon CloudFront: Provides global distribution for static content and acts as the front-end for the REST API.
   - Amazon S3: Stores static website files, distributed globally via CloudFront. Access is secured using Origin Access Control (OAC), ensuring only CloudFront can access the S3 bucket.
   - Amazon API Gateway & AWS Lambda: Form the serverless REST API 
   - Amazon DynamoDB: Stores dynamic blog data.
   - DAX Caching layer: Caches reads on DynamoDB for improved performance.
   - DynamoDB Global Tables: Leveraged to serve data globally, ensuring low latency for users worldwide. (Note: Aurora Global Database could also be used - but in this case it wouldn't have been serverless, it would've been provisioned Aurora).
   - DynamoDB Stream: Triggers an AWS Lambda function upon changes (e.g., new user subscription).
   - Amazon Simple Email Service (SES): Used by a Lambda function (with an appropriate IAM Role) to send welcome emails to new subscribers.
   - S3 Triggers (Lambda, SQS, SNS): S3 can trigger a Lambda function for events like photo uploads, facilitating thumbnail generation. SQS (Simple Queue Service) or SNS (Simple Notification Service) can optionally be used for notifications.

- Key Patterns:
   - Static Content Distribution: Achieved globally and securely with CloudFront and S3, using OAC.
   - Dynamic Content: A serverless REST API built with API Gateway, Lambda, and DynamoDB.
   - Event-Driven Flows: DynamoDB Streams trigger Lambda for user emails, and S3 events trigger Lambda for image processing (like thumbnail generation).

![](https://github.com/DanialArab/images/blob/main/AWS/thumbnail_gen_flow.png)

<a name="45"></a>
### Micro Services Architecture 
This section discusses the adoption of a microservices architecture, where many services interact via REST APIs, and each service can have a varied internal design. The goal is a leaner development lifecycle (We want each service to scale independently).

- Common Components/Patterns:
   - Synchronous Patterns: Amazon API Gateway and Elastic Load Balancing are used for direct, real-time interactions between services or with clients.
   - Asynchronous Patterns: SQS, Kinesis, SNS, and Lambda triggers (e.g., from S3) enable decoupled and non-blocking communication between services.
   - Other Services: The environment can include AWS Lambda, ElastiCache, Amazon EC2 Auto Scaling, Amazon RDS, Amazon ECS, DynamoDB, and Amazon Route 53.

- Challenges and Serverless Solutions:
     - Challenges: Microservices can introduce challenges such as repeated overhead for creating new services, difficulty optimising server density/utilisation, complexity of managing multiple service versions, and increased client-side code requirements for integration.
     - Serverless Solutions: API Gateway and Lambda help overcome some of these challenges by providing automatic scaling and a pay-per-usage model. They also facilitate easy cloning of APIs and reproduction of environments, along with generated client SDKs through Swagger integration for the API Gateway.

![](https://github.com/DanialArab/images/blob/main/AWS/micro_service_env.png)

<a name="46"></a>
### Software Updates Offloading (Optimising Existing EC2 Application)

This architecture addresses a common problem: an application running on EC2 that experiences high costs and CPU usage during mass software update distributions. The objective is to optimise costs without changing the existing application code.

- Problem Statement: An application running on EC2 instances within an Auto Scaling Group across multiple Availability Zones, backed by Amazon Elastic File System, becomes costly and resource-intensive when distributing large software updates due to a surge in network requests.
- Solution: Introduce Amazon CloudFront in front of the existing EC2 application.
- Why CloudFront Works:
     - No Architectural Changes: Crucially, this solution requires no changes to the existing application architecture .
     - Edge Caching: CloudFront caches the static software update files at edge locations globally . Since software updates are static and do not change, they are ideal for caching .
     - Serverless Scaling: While the EC2 instances are not serverless, CloudFront is serverless and scales automatically to handle the demand .
     - Cost and Resource Savings: By offloading distribution to CloudFront, the EC2 Auto Scaling Group will not need to scale as much, leading to significant savings in EC2 costs, network bandwidth costs, and improved availability. This makes the existing application more scalable and cheaper .

![](https://github.com/DanialArab/images/blob/main/AWS/massive_sortware_update.png)

--------------------------------------------------------------------------------
Think of these architectures like different types of delivery services.
- The MyTodoList Mobile App is like a specialised local courier service. It handles authenticating customers, letting them access their specific storage lockers (S3), processing their requests (Lambda/DynamoDB), and making sure the most frequently requested items are kept ready in a quick access bin (DAX/API Gateway Cache) for speedy delivery.
- The MyBlog.com Website is a global content delivery network. It uses a vast network of warehouses (CloudFront edge locations) to quickly serve static content from a central storage facility (S3) worldwide. For dynamic requests, it uses automated order processing (API Gateway/Lambda) and keeps track of stock globally (DynamoDB Global Tables). It also has automated systems for new customer greetings (SES via DynamoDB Stream) and photo processing (S3 trigger to Lambda for thumbnails).
- The Microservices Architecture is like a collection of many independent, specialised shops, each handling a specific part of a larger business. They can communicate directly (synchronous) or through a messaging system (asynchronous) and each shop can be set up in its own unique way. Serverless services like API Gateway and Lambda act as self-managing, instantly scalable storefronts for these shops, reducing the setup hassle.
- The Software Updates Offloading is like putting a global distribution centre (CloudFront) in front of an existing factory (EC2 application). Instead of the factory directly shipping every single product update to every customer, the distribution centre receives one copy, then efficiently delivers it to countless customers from its local depots, significantly reducing the factory's workload and shipping costs, without changing how the factory operates.


<a name="47"></a>
# Containers on Cloud

Docker is a software development platform that allows applications to be packaged into containers. These containers can run on any operating system, ensuring predictable behaviour and fewer compatibility issues, making applications easier to maintain and deploy. Docker is suitable for various use cases, including microservices architecture and "lift-and-shift" migrations of on-premises applications to the AWS cloud.

Docker vs. Virtual Machines (VMs) While Docker is a form of virtualisation, it differs from traditional VMs. Docker containers share resources with the host operating system, enabling many containers to run on a single server. In contrast, VMs typically have their own guest operating systems, hypervisors, and applications. To get started with Docker, you typically use a Dockerfile to build an image, which then runs as a container.
Docker Image Storage Docker images are stored in **Docker Repositories**. Key repositories include:

- Docker Hub: A public repository where you can find base images for many technologies and operating systems (e.g., Ubuntu, MySQL).
- Amazon ECR (Amazon Elastic Container Registry): AWS's own container registry, which can be private or public (Amazon ECR Public Gallery). ECR is fully integrated with Amazon ECS, backed by Amazon S3, and access is controlled via IAM. It also supports image vulnerability scanning, versioning, and lifecycle management. Images are pushed to ECR and pulled by container services.

<a name="48"></a>
## Docker Containers Management on AWS 

AWS provides several services for managing Docker containers:
- Amazon Elastic Container Service (Amazon ECS): Amazon's own container platform.
- Amazon Elastic Kubernetes Service (Amazon EKS): Amazon's managed Kubernetes service, which is an open-source system for automating deployment, scaling, and management of containerised applications.
- AWS Fargate: Amazon's serverless container platform that works with both ECS and EKS.
- Amazon ECR: Used to store container images.

<a name="49"></a>
## ECS  
Amazon ECS (Elastic Container Service) ECS allows you to launch Docker containers on AWS by launching ECS Tasks on ECS Clusters. It offers two main launch types:
- EC2 Launch Type:
    - You are responsible for provisioning and maintaining the underlying EC2 instances (infrastructure).
    - Each EC2 instance must run the ECS Agent to register with the ECS Cluster.
    - AWS handles starting and stopping containers.
- Fargate Launch Type:
     - Serverless: You do not provision or manage any EC2 instances.
     - You simply define task definitions, and AWS runs ECS Tasks based on your specified CPU and RAM requirements.
     - Scaling is simplified by increasing the number of tasks.

IAM Roles for ECS:

- EC2 Instance Profile (for EC2 Launch Type only): Used by the ECS agent for API calls to the ECS service, sending logs to CloudWatch Logs, pulling Docker images from ECR, and referencing sensitive data in Secrets Manager or SSM Parameter Store.
- ECS Task Role: Allows each task to have its own specific permissions, defined in the task definition, enabling different ECS services to use different roles.

Load Balancer Integrations:

- Application Load Balancer (ALB) is supported for most use cases.
- Network Load Balancer (NLB) is recommended for high throughput/performance or when paired with AWS PrivateLink.
- Classic Load Balancer (CLB) is supported but not recommended, lacking advanced features and Fargate compatibility.

Data Volumes (EFS):
• You can mount Amazon EFS (Elastic File System) file systems onto ECS tasks, which works for both EC2 and Fargate launch types.
• Tasks in any Availability Zone can share the same data in the EFS file system.
• Combining Fargate with EFS provides a serverless solution for persistent, multi-AZ shared storage. Note that Amazon S3 cannot be mounted as a file system.

ECS Service Auto Scaling:
• Automatically increases or decreases the desired number of ECS tasks.
• Uses AWS Application Auto Scaling and can scale based on metrics such as:
    ◦ ECS Service Average CPU Utilisation.
    ◦ ECS Service Average Memory Utilisation.
    ◦ ALB Request Count Per Target.
• Scaling policies include Target Tracking (based on a target value for a CloudWatch metric), Step Scaling (based on a CloudWatch Alarm), and Scheduled Scaling (for predictable changes at specific dates/times).
• ECS Service Auto Scaling (task level) is distinct from EC2 Auto Scaling (EC2 instance level). Fargate Auto Scaling is simpler due to its serverless nature.

EC2 Launch Type – Auto Scaling EC2 Instances:
• To accommodate ECS Service Scaling, you can add underlying EC2 instances.
• This is achieved through Auto Scaling Group (ASG) Scaling (e.g., based on CPU utilisation).
• Alternatively, an ECS Cluster Capacity Provider can automatically provision and scale the infrastructure (EC2 instances) for your ECS Tasks by adding EC2 instances when capacity (CPU, RAM) is low.

ECS Task Invocation:
• ECS tasks can be invoked by Amazon EventBridge, either in response to events (e.g., S3 object upload) or on a schedule (e.g., for batch processing).
• Tasks can also poll messages from an SQS Queue.
• EventBridge can also be used to intercept events when ECS tasks stop, allowing for notifications or automated actions.

<a name="50"></a>
## EKS

Amazon EKS (Elastic Kubernetes Service) EKS is a managed Kubernetes service on AWS.
• It's an alternative to ECS with a similar goal but a different API.
• EKS supports deploying worker nodes on EC2 instances or serverless containers using Fargate.
• It's particularly useful if your organisation already uses Kubernetes on-premises or in another cloud and wishes to migrate to AWS. Kubernetes is cloud-agnostic.
• For multi-region deployments, you would deploy one EKS cluster per region.
• Logs and metrics can be collected using CloudWatch Container Insights.

EKS Node Types:
• Managed Node Groups: EKS creates and manages the EC2 instances (nodes) for you, as part of an ASG managed by EKS. Supports On-Demand or Spot Instances.
• Self-Managed Nodes: You create and register the nodes to the EKS cluster and manage them with an ASG. You can use pre-built Amazon EKS Optimized AMIs. Supports On-Demand or Spot Instances.
• AWS Fargate: No maintenance required, as no nodes are managed by you.
EKS Data Volumes:
• EKS requires a StorageClass manifest and leverages a Container Storage Interface (CSI) compliant driver.
• It supports various storage solutions, including Amazon EBS, Amazon EFS (which works with Fargate), Amazon FSx for Lustre, and Amazon FSx for NetApp ONTAP.

AWS App Runner App Runner is a fully managed service that simplifies the deployment of web applications and APIs at scale.
• It requires no infrastructure experience.
• You can start with either your source code or a container image (Docker).
• App Runner automatically builds and deploys the web application, handling automatic scaling, high availability, load balancing, and encryption.
• It also supports VPC access and connectivity to databases, caches, and message queues.
• Use cases include web applications, APIs, microservices, and rapid production deployments.



**Reference:**

AWS Neptune:

https://www.youtube.com/playlist?list=PLd13Ui6RDb8m5X15ZrQd8XuOXV6RZbPSO 
https://docs.aws.amazon.com/neptune/latest/userguide/feature-overview-db-clusters.html
https://www.youtube.com/watch?v=Y42OwmUF23s&list=PLd13Ui6RDb8m5X15ZrQd8XuOXV6RZbPSO&index=5

