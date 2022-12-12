## Deploying Blinkenlights on AWS Fargate 

We will walk through an example of deploying Blinkenlights in an Amazon Virtual Private Cloud (VPC) on Amazon Elastic Container Service (ECS) and Amazon Fargate. This example assumes Blinkenlights has a public IP address (for internet access) and guides you through a complete setup. Your environment may be a simpler, or a more complex, version of this deployment.

You will need the Amazon ECS command line interface (CLI) to deploy Blinkenlights. To install the Amazon ECS CLI, see [ECS CLI installation](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_installation.html).

### Update parameters

*   Update the `ecs-params.yml` file for your desired environment
*   Update the `docker-compose.yml` file and substitute the parameters as needed
    *   Update the Blinkenlights license `BL_LICENSE`
    *   Set the logging to point to the correct region (awslog-region). The default is "us-east-1". This will create a cluster and service running on ECS that has "[Service Discovery](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-discovery.html)". Service Discovery means Blinkenlights is discoverable by services on the same VPC. 

### Create environmental variables
Update and add these environmental variables to help make the setup process easier
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
export AWS_SECURITY_GROUP_NAME=blinkenlights-sg
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

### Updating the docker-compose.yml

```
ecs-cli compose --project-name $AWS_SERVICE --region $AWS_REGION create --launch-type FARGATE --profile $AWS_PROFILE
```
```
aws ecs update-service --cluster $AWS_CLUSTER --service $AWS_SERVICE --force-new-deployment
```

### Updating and restarting Blinkenlights on AWS

_If you already have a Blinkenlights service:_
```
aws ecs update-service --cluster $AWS_CLUSTER --service $AWS_SERVICE --force-new-deployment 
```

For Blinkenlights, the command would look like this:

```
aws ecs update-service --cluster blinkenlights --service blinkenlights --force-new-deployment 
```
_If you do not have a Blinkenlights service:_
```
aws ecs update-service --cluster $AWS_CLUSTER --service $AWS_SERVICE
```
For Blinkenlights, the command would look like this:
```
aws ecs update-service --cluster blinkenlights --service blinkenlights
```

