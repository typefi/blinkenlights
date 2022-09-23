## Deploying Blinkenlights on AWS Fargate 

We will walk through an example of deploying Blinkenlights in an Amazon Virtual Private Cloud (VPC) on Amazon Elastic Container Service (ECS) and Amazon Fargate. This example assumes Blinkenlights has a public IP address (for internet access) and guides you through a complete setup. Your environment may be a simpler, or a more complex, version of this deployment.

You will need the Amazon ECS command line interface (CLI) to deploy Blinkenlights. To install the Amazon ECS CLI, see [ECS CLI installation](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_installation.html).

### Update parameters

*   Rename `ecs-params-sample-fargate.yml`to `ecs-params.yml`and update the `ecs-params.yml` file for your desired environment
*   Rename `docker-compose-sample-fargate.yml` to `docker-compose.yml` and update the `docker-compose.yml` file and substitute the parameters as needed
    *   Point to your remote remote mongodb database `NODE_CONFIG`. Don't have MongoDB? Try [MongoDB Atlas](https://www.mongodb.com/cloud/atlas). The free tier will suffice
    *   Update the Blinkenlights license `BL_LICENSE`
    *   Set the logging to point to the correct region (awslog-region). The default is "us-east-1". This will create a cluster and service running on ECS that has "[Service Discovery](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-discovery.html)". Service Discovery means Blinkenlights is discoverable by services on the same VPC. 

### Create environmental variables
```
export AWS_REGION=us-east-1
export AWS_ACCESS_KEY_ID=
export AWS_SECRET_ACCESS_KEY=
export AWS_VPC_ID=
export AWS_SUBNET_ID_1=
export AWS_SUBNET_ID_2=
export AWS_PROFILE=default
export AWS_ROLE=ecsTaskExecutionRole
export AWS_NAMESPACE-ID=blinkenlights-demo
export AWS_SECURITY_GROUP_NAME=blinkenlights-sg2
export AWS_SECURITY_GROUP_CIDR=0.0.0.0/0
export AWS_CLUSTER=blinkenlights
export CLI_CLUSTER_CONFIG=blinkenlights-config
export ECS_PROFILE=blinkenlights-profile
export AWS_SERVICE=blinkenlights-service

```

### Start Blinkenlights on AWS

 **1)** _This step is optional and not necessary in all cases._ **Permit access only from trusted private IP ranges**. While Blinkenlights requires public IP access in AWS Fargate, we suggest permitting access only from trusted private IP ranges. However, this poses several challenges with [AWS Service Discovery](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-discovery.html). To fix this, we suggest using an [Amazon Route 53](https://aws.amazon.com/route53/) registered domain name and creating a [public namespace based on DNS](https://docs.aws.amazon.com/cloud-map/latest/api/API_CreatePublicDnsNamespace.html) for API calls and public DNS queries. 
Visit [Cloud Map in the AWS console](https://console.aws.amazon.com/cloudmap/home) to get started.

**2) Create and attach the task execution role**
```
aws iam --region $AWS_REGION create-role --role-name $AWS_ROLE --assume-role-policy-document file://task-execution-assume-role.json --profile $AWS_PROFILE
```
```
aws iam --region $AWS_REGION attach-role-policy --role-name $AWS_ROLE --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy --profile $AWS_PROFILE
```
**3) Create a cluster config**
```
ecs-cli configure --cluster $AWS_CLUSTER --default-launch-type FARGATE --config-name $CLI_CLUSTER_CONFIG --region $AWS_REGION
```
**4) Create a CLI profile**
```
ecs-cli configure profile --access-key $AWS_ACCESS_KEY_ID --secret-key $AWS_SECRET_ACCESS_KEY --profile-name $ECS_PROFILE
```
**5) Create the security group**
```
aws ec2 create-security-group --group-name $AWS_SECURITY_GROUP_NAME --description "blinkenlights security group" --vpc $AWS_VPC_ID --profile $AWS_PROFILE
```
This command outputs the `AWS_SECURITY_GROUP_ID`

**6) Set security group inbound rules**

_HTTP/HTTPS_
```
aws ec2 authorize-security-group-ingress --group-id $AWS_SECURITY_GROUP_ID --protocol tcp --port 80 --cidr $AWS_SECURITY_GROUP_CIDR --profile $AWS_PROFILE
```

```
aws ec2 authorize-security-group-ingress --group-id $AWS_SECURITY_GROUP_ID --protocol tcp --port 443 --cidr $AWS_SECURITY_GROUP_CIDR --profile $AWS_PROFILE
```
_Websocket_
```
aws ec2 authorize-security-group-ingress --group-id $AWS_SECURITY_GROUP_ID --protocol tcp --port 8081 --cidr $AWS_SECURITY_GROUP_CIDR --profile $AWS_PROFILE
```
**7) Create a cluster**
```
ecs-cli up --cluster-config $CLI_CLUSTER_CONFIG --ecs-profile $ECS_PROFILE --vpc $AWS_VPC_ID --subnets $AWS_SUBNET_ID_1, $AWS_SUBNET_ID_2 --security-group $AWS_SECURITY_GROUP_ID --profile $AWS_PROFILE
```
**8) Update ECS configuration**

Update the `ecs-params.yml` file with the subnet IDs and security group ID

**9) Deploy service to cluster**
```
ecs-cli compose --ecs-params ecs-params.yml --project-name $AWS_SERVICE service up --vpc $AWS_VPC_ID --cluster-config $CLI_CLUSTER_CONFIG --ecs-profile $ECS_PROFILE
```
```
aws ecs update-service --cluster $AWS_CLUSTER --service $AWS_SERVICE --force-new-deployment --platform-version "1.3.0" --force-new-deployment --query 'service.platformVersion'
```

### Stop Blinkenlights on AWS

**1) Remove service**
```
ecs-cli compose --project-name $AWS_SERVICE service down --cluster-config $CLI_CLUSTER_CONFIG --ecs-profile $ECS_PROFILE
```
**2) Remove cluster**
```
ecs-cli down --cluster-config $CLI_CLUSTER_CONFIG --ecs-profile $ECS_PROFILE
```

### Updating and restarting Blinkenlights on AWS

_If you already have a Blinkenlights service:_
```
aws ecs update-service --cluster CLUSTER_NAME --service SERVICE_NAME --force-new-deployment 
```

For Blinkenlights, the command would look like this:

```
aws ecs update-service --cluster blinkenlights --service blinkenlights --force-new-deployment 
```
_If you do not have a Blinkenlights service:_
```
aws ecs update-service --cluster CLUSTER_NAME --service SERVICE_NAME
```
For Blinkenlights, the command would look like this:
```
aws ecs update-service --cluster blinkenlights --service blinkenlights
```

## **Using Blinkenlights**
### User interface

The main view in Blinkenlights is called the _Dashboard_. The Dashboard shows your job queues, any active or queued jobs and their elapsed or wait times, and your InDesign Servers instances and their status.

On the left side of the screen is the controls menu, which provides quick access to the administrative functions you can perform. The menu options are:
*   Search
*   Add server
*   Add queue
*   Metrics
*   Test lab
*   Settings

At the bottom of the screen is a collapsed _Logs_ tab that provides an at-a-glance view of completed jobs and job warnings or errors. Clicking anywhere on this tab causes it to “slide up” to give you complete access to the detailed logs.

### Creating your first job queue

Click **Add queue** to create your first job queue. By default, new job queues are named "New queue" followed by a unique identifier. For example, "New queue (5c65e41e2d4376770020b9a8)". Enter a new name and then click out of the field to save your changes.

Follow these conventions when naming a job queue:
*   Job queue names must be unique.
*   You can use numbers and most symbols.
*   The name cannot be longer than 255 characters.
*   Avoid using [control characters](https://en.wikipedia.org/wiki/Control_character) or any of the following: * &lt; > : " / | \ ? ; ! ^
*   Do not start or end a name with a period (.).
*   Avoid starting or ending a name with a space.

After creating your first job queue, add one or more InDesign Server instances.

### Adding an InDesign Server

Click **Add server** to open the _Add InDesign Server_ dialogue.

1. Enter the Hostname and Port for your InDesign Server instance. For example, Hostname: ourhost.company.com and Port: 18383.
2. Optionally enter a Nickname to help you more quickly identify a server (note: nicknames must be unique) and assign it to a job queue (newly added InDesign Server instances are automatically added to the unassigned job queue).

**Note**: You can also move InDesign Server instances between job queues by dragging the server into a new queue.

**Note**: A single InDesign Server instance cannot be added to more than one queue.


### Searching or filtering jobs

The Search tab is located at the top right of the screen. You can hide or show the Search tab by clicking the Search control on the left of the screen.

You can search or filter jobs using wildcard searches. Both active and queued jobs that match your search criteria will be highlighted, while non-matching jobs will be dimmed. Clear your search to restore the normal appearance.

You can also use the Search tab to control which job metadata (Customer, Owner, Origin, Description, or Id) is displayed as a job label.


### Viewing job details

Click on any job in the Dashboard to view its details, such as its priority, status, position in the queue, when it was submitted to Blinkenlights, and how long it's been in the queue, and to select metadata (Customer, Owner, Origin, and Description) from the job submission that can be used to help you track and identify jobs as they are queued or in progress.


### Cancelling a job

To cancel a job, click on the job in the Dashboard to view its details, and then click Cancel. Cancelling a job will remove it from the queue and notify the submitter that the job was cancelled.


### Configuring an InDesign Server

Click an instance of InDesign Server to open the Edit InDesign Server dialog. Use the Edit InDesign Server dialog to make changes to the server hostname, port, or nickname, or to Disable, Unlock, or Remove the server.

Note: Changes to the hostname, port, or nickname won't apply until you click Save.

# Configue Typefi Server for Workgroup
Typefi Server for Workgroup is generally configured from Admin > External Applications > InDesign.  Time to change that by going to Admin > External Applications > Blinkenlights and checking the box, supplying the http://<FQDN> or http://<IP> path and leaving the port blank assuming the config is on :80.  After the values are set, Refresh Queue will populate with queues from Blinkenlights and workflows may need an update.  The setting can be disabled for troubleshooting. 
