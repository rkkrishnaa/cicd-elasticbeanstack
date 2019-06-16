Architecture:
-------------

![architecture](https://s3.amazonaws.com/opensource-articles/aws/cicd_beanstack.png?raw=true)

Building an Immutable Infrastructure is an ultimate goal of this solution. Reusability of code for creating a similar environment in a short duration of time and more developer friendly is an another aspect of this solution. Cloudformation is the orchestrator for provisioning and maintaining the infrastructure through infrastructure as code. The entire infrastructure can be created by executing a single template. It will create a nested stack with all dependent resources. The life cycle of each components of a stack can be managed by updating parent stack. It will detect for the changes from all nested templates and execute the change sets.

We use Cloudformation, VPC, EC2, ELB, S3, Autoscaling, Elastic Beanstack, Code Commit, Code Pipeline, SNS, IAM for implementing this solution. Cloudformation is the core component of the infrastructure which maintains the state of all components. Our network infrastructure leverages VPC and its components for building a secured network on top of AWS. A single VPC spans across all availability zones of a region with different subnets to ensure the servers are distributed across availability zones for building a highly available and fault tolerant infrastructure. We have a different subnet for different tiers of a web application. Our application is designed in a two tier architecture pattern. Application logic is implemented in a EC2 server managed by Elastic Beanstack and Data tier is implemented in RDS. Both tiers are scalable.

For infrastructure administration and maintenance, Bastion host is deployed in a public subnet. It is a highly secured and created from a prebuilt ami provided by AWS. It will allow ssh connection only from a trusted ip source. Application servers and Database servers are hosted in a private subnets. It can be only accessed from Bastion host. Servers can be connected only by key pair authentication to avoid vulnerabilities. App server can access internet through NAT gateway for software installation.

Classic Elastic load balancer is a user facing component to accept the web requests in our application. These traffic are routed to the backend EC2 servers. Backend server takes care of processing the web request and return the response to ELB which is then consumed by the end user. ELB is deployed in a public subnet and it is secured by a VPC security group which will allow only http/https inbound traffic from external sources. ELB will only access the back end servers either by http/https protocol. To ensure high availability and uniform distribution of traffic, we have enabled cross zone load balancing. Apart from that, we have configured load balancer to support session persistence, maintaining idle timeout between the load balancer and the client.

We use RDS Aurora database as a database tier for the application. It is deployed as a cluster with read/write endpoints. Both servers and database instances are secured by strong security group policy to avoid access from untrusted network source. 

AWS Code commit is a source code repository for this project, It is a highly available, private repository managed by AWS.  S3 bucket is used for storing the artifacts. This artifact is used by code pipeline to deploy it on different environment. 

CI/CD pipeline is the core component of this project which builds the code and deploy the changes to the server. We use AWS Code Pipeline for buiilding CI/CD pipeline.

How to create the infrastructure?
---------------------------------
Our infrastructure is created and managed by AWS Cloudformation. Before executing the template, please follow the below instructions to create an infrastructure.

Pre-requisites:
---------------
1.     CodeCommit Repository with source code of the application
2.     SNS topic with email subscribers
3.     S3 bucket containing cloudformation templates, Create a folder called �templates� indide a bucket and upload the cloudformation templates into that folder.

Steps:
------
1. Log in to the AWS Management Console and select CloudFormation in the Services menu.
 
2. Click Create Stack. This is the only option if you have a currently running stack.

3. Enter the required input parameters and execute the stack. The order of execution of stack is given below.
Cloudformation template parses the inputs and resource section of a parent stack. Initially, it will create a network stack for the infrastructure. It includes VPC, Subnet, Routetable, Nacl, Internet Gateway, NAT Gateway, Routing policy.
	Bastion host is created with an appropriate security group policy. 
	Elastic Beanstack application will be created for deploying different environments such as dev, staging and production.
	Aurora Database cluster will be created in the next step for dev, staging and production environment. DB server has its own security group to control the inbound access. It has its own parameter group  as well as config group.
	Elastic beanstack application environment will be created for different environments. Here, our runtime is PHP and we have created a configuration group with the required parameters such as load balancer configuration, EC2 autoscaling configuration, environment variables for application environments.
	Continuous Integration and Delivery pipe will be created at the last step. It uses code commit as source and apply changes to the elastic beanstack environment whenever there is a change in the source code with manual approval in staging and production environment. Our template will create a required IAM roles for code pipeline project.

4. After few minutes, the stack is available and we can access the services. Initially, Code pipeline releases the changes to the instances hosted in the elastic beanstack environment.
5. Access the environment UI and check the application.
6. Update some changes in the source code, CI job will be triggered within a minute, It will pull the source code from the code commit repo and waiting for a manual approval in staging and prod env to apply the changes to the server, Elastic beanstack will create a new resources and the code is deployed in the environment. Then it will remove the old resource after successful deployment. This action continues whenever the new version is committed to the repo.

CI/CD pipeline for deploying a PHP application hosted in Elastic beanstack envionment:
-------
Continuous integration (CI) is a software development practice where developers regularly merge their code changes into a central repository, after which automated builds and tests are run. The key goals of CI are to find and address bugs more quickly, improve software quality, and reduce the time it takes to validate and release new software updates. In our case, we have built a CI pipeline using AWS Code Commit and Code Pipeline. It has two three stages.

Stage1: Source
--------------
When the pipeline is triggered by a change in the configured repo branch, the very first step is to download the source code from the server to the workspace where the next set of actions to be done. Here, we have configured the pipeline to pull the specified repository name and branch. If a single stage has multiple actions, then we can mention run order to execute a particular action in some sequences.

Stage2: Approve
---------------
Some projects might not requires build and we can move to the next stage. In our case, it is approval stage. Project manager can approve the changes to be deployed in the environment or deny the changes. We use SNS for sending notification to the subscribers to approve the changes. If the action is approved, pipe will move to the next stage otherwise it will be aborted.

Stage3: Deploy
--------------
Depending upon the approval, the pipeline may or may not reach the deploy stage. During deploy stage the code is deployed in all the application environments. Elastic beanstack�s deployment strategy high endorses Blue-Green deployment pattern. During deployment, users can access the application with older version. No changes will be done in the existing servers. Beanstack creates a new set of resources and apply the changes to the server. After successful deployment, the latest version of the application can be accessed by the users and the old servers are removed.

The basic challenges of implementing CI include more frequent commits to the common codebase, maintaining a single source code repository, automating builds, and automating testing. Additional challenges include testing in similar environments to production, providing visibility of the process to the team, and allowing developers to easily obtain any version of the application.

Continuous delivery (CD) is a software development practice where code changes are automatically built, tested, and prepared for production release.  Continuous delivery can be fully automated with a workflow process or partially automated with manual steps at critical points. 

With continuous deployment, revisions are deployed to a production environment automatically without explicit approval from a developer, making the entire software release process automated. 
