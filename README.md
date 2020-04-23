# blinkenlights
Typefi Blinkenlights is a load balancer and queue management system for Adobe InDesign Server.


Updating/Restarting
aws ecs update-service --cluster CLUSTER_NAME --service SERVICE_NAME --force-new-deployment
aws ecs update-service --cluster CLUSTER_NAME --service SERVICE_NAME

aws ecs update-service --cluster blinkenlights --service blinkenlights --force-new-deployment
aws ecs update-service --cluster blinkenlights --service blinkenlights



Key steps in deploying Blinkenlights.


We provide instructions on deploying Blinkenlights in an Amazon VPC on Amazon ECS/Fargate.  The example assumes Blinkenlights has a public IP address (for interent access) and guides you through a complete setup.  Your environment may be a simpler, or a more complex, version of this deployment.  
Update the docker-compose.yml file and substitute the parameters as required.
Update the ecs-params.yml file for your desired environment.


* Point to your remote remote mongodb database.  Don't have MongoDB? Try MongoDB Atlas. Free tier will suffice.
* Set the logging to point to the correct region. Defaults to "us-east-1"
* This will create a cluster & service running on ECS that has "service discovery"
Should be discoverable by services on the same VPC using this endpoint: "blinkenlights.blinkenlights"
* You can find it in ECS -> Clusters -> blinkenlights -> tasks -> `task` -> public and private IP.


## Start Blinkenlights on AWS
* This step is optional and not necessary in all cases.
While Blinknenlights requires public IP access in AWS Fargate, we suggest permitting only access from trusted private IP ranges.  This poses several challenges with Service Discovery.  We suggest using a Route53 registered domain name and creating a public-dns-namespace for `API calls and public DNS queries`. Visit https://console.aws.amazon.com/cloudmap/home to get started.
* install ecs-cli
  * https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_installation.html
* Create task execution role
  * aws iam --region us-east-1 create-role --role-name ecsTaskExecutionRole --assume-role-policy-document file://task-execution-assume-role.json
  * aws iam --region us-east-1 attach-role-policy --role-name ecsTaskExecutionRole --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
* Create a cluster config
  * ecs-cli configure --cluster blinkenlights --default-launch-type FARGATE --config-name blinkenlights --region `AWS_REGION`
* Create a cli profile
  * ecs-cli configure profile --access-key `AWS_ACCESS_KEY_ID` --secret-key `AWS_SECRET_ACCESS_KEY` --profile-name blinkenlights-profile
* Create security group
  * aws ec2 create-security-group --group-name `blinkenlights-sg` --description "blinkenlights security group"
    * outputs the `AWS_SECURITY_GROUP_ID`
* Set sectury group inbound rules
  * HTTP
    * aws ec2 authorize-security-group-ingress --group-id `AWS_SECURITY_GROUP_ID` --protocol tcp --port 80 --cidr `RANGE`
    * aws ec2 authorize-security-group-ingress --group-id `AWS_SECURITY_GROUP_ID` --protocol tcp --port 443 --cidr `RANGE`
  * Websocket
    * aws ec2 authorize-security-group-ingress --group-id `AWS_SECURITY_GROUP_ID` --protocol tcp --port 8081 --cidr `RANGE`
* Create a cluster
  * ecs-cli up --cluster-config blinkenlights --ecs-profile blinkenlights-profile --vpc `AWS_VPC_ID` --subnets `AWS_SUBNET_ID_1`, `AWS_SUBNET_ID_2` --security-group `AWS_SECURITY_GROUP_ID`
* Update ecs config
  * update the `ecs-params.yml` file with the subnet ids & security group id
* Deploy service to cluster
  * ecs-cli compose --project-name blinkenlights service up --private-dns-namespace blinkenlights --vpc `AWS_VPC_ID` --create-log-groups --cluster-config blinkenlights --ecs-profile blinkenlights-profile --enable-service-discovery

  or

  * ecs-cli compose --ecs-params ecs-params.yml --project-name blinkenlights service up --vpc vpc-5190d138 --cluster-config blinkenlights --ecs-profile blinkenlights-profile --enable-service-discovery --public-dns-namespace-id `NAMESPACE-ID`

## Stop Blinkenlights on AWS
* Remove service
  * ecs-cli compose --project-name blinkenlights service down --cluster-config blinkenlights --ecs-profile blinkenlights-profile
* Remove cluster
  * ecs-cli down --cluster-config blinkenlights --ecs-profile blinkenlights-profile
