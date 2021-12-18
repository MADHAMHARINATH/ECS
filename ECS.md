# How To Create An ECS Cluster Using AWS CLI:
- Prerequisite: AWS CLI installed and configured with proper access.
## Step-1 : Create a cluster with fargate capacity provider
```
aws ecs create-cluster \
--cluster-name myecscluster \
--capacity-providers FARGATE FARGATE_SPOT
```
## Step-2 : Create a task definition to run tasks and services in your ECS Cluster.
- copy the below code to myecsclustertaskdef.json
```
vi myecsclustertaskdef.json
```
```
{
    "family": "mytaskdef", 
    "networkMode": "awsvpc", 
    "containerDefinitions": [
        {
            "name": "myapp", 
            "image": "httpd:2.4", 
            "portMappings": [
                {
                    "containerPort": 80, 
                    "hostPort": 80, 
                    "protocol": "tcp"
                }
            ], 
            "essential": true, 
            "entryPoint": [
                "sh",
        "-c"
            ], 
            "command": [
                "/bin/sh -c \"echo 'hello from ecs fargate cluster' >  /usr/local/apache2/htdocs/index.html && httpd-foreground\""
            ]
        }
    ], 
    "requiresCompatibilities": [
        "FARGATE"
    ], 
    "cpu": "256", 
    "memory": "512"
}
```
- enter :wq to come out from the vi editor.
- Register the task definition.
```
aws ecs register-task-definition \
--cli-input-json file://myecsclustertaskdef.json
```
## Step-3 : Create a service using the task definition created in step 2.
- Get your default vpc subnet and sg info
```
AWS_VPC_ID=$(aws ec2 describe-vpcs \
--filters "Name=isDefault, Values=true" \
--query 'Vpcs[0].VpcId' \
--output text) &&
AWS_DEFAULT_SECURITY_GROUP_ID=$(aws ec2 describe-security-groups \
--filters "Name=vpc-id,Values=$AWS_VPC_ID" \
--query 'SecurityGroups[?GroupName == `default`].GroupId' \
--output text) &&
AWS_SUBNET_ONE_ID=$(aws ec2 describe-subnets \
--filters "Name=vpc-id,Values=$AWS_VPC_ID" \
--query 'Subnets[?AvailabilityZone == `ap-south-1a`].SubnetId' \
--output text)
```
- Create security group ingress rules.
```
aws ec2 authorize-security-group-ingress \
--group-id $AWS_DEFAULT_SECURITY_GROUP_ID \
--ip-permissions '[{"IpProtocol": "tcp", "FromPort": 22, "ToPort": 22, "IpRanges": [{"CidrIp": "0.0.0.0/0", "Description": "Allow SSH"}]}]' &&
aws ec2 authorize-security-group-ingress \
--group-id $AWS_DEFAULT_SECURITY_GROUP_ID \
--ip-permissions '[{"IpProtocol": "tcp", "FromPort": 80, "ToPort": 80, "IpRanges": [{"CidrIp": "0.0.0.0/0", "Description": "Allow HTTP"}]}]'
```
- Create a service in the ecs cluster using task definition
```
aws ecs create-service \
--cluster myecscluster \
--service-name myservice \
--task-definition mytaskdef:1 \
--desired-count 1 \
--launch-type "FARGATE" \
--network-configuration "awsvpcConfiguration={subnets=[$AWS_SUBNET_ONE_ID],securityGroups=[$AWS_DEFAULT_SECURITY_GROUP_ID],assignPublicIp=ENABLED}"
```
## Step-4 : Get details of your ECS Cluster using AWS CLI.
- List all available ecs clusters
```
aws ecs list-clusters
```
- Get the details of ecs cluster
```
aws ecs describe-clusters \
--cluster myecscluster
```
- Get the details of ecs cluster capacity provider
```
aws ecs describe-capacity-providers
```
- List all available task definitions
```
aws ecs list-task-definitions
```
- Get the details of ecs cluster task definition
```
aws ecs describe-task-definition \
--task-definition mytaskdef:1
```
- List all the available cluster services
```
aws ecs list-services \
--cluster myecscluster
```
- Get the details of ecs cluster service
```
aws ecs describe-services \
--cluster myecscluster \
--services myservice
```
- List all the available cluster services
```
aws ecs list-services \
--cluster myecscluster
```
- Get the details of ecs cluster service
```
AWS_ECS_TASK_ARN=$(aws ecs list-tasks \
--cluster myecscluster \
--query 'taskArns' \
--output text) &&
aws ecs describe-tasks \
--cluster myecscluster \
--tasks $AWS_ECS_TASK_ARN
```
## Step-5 : Check your application running in the ECS Cluster.
- Get public ip of your ecs deployed application
```
AWS_ECS_FARGATE_ENI=$(aws ecs describe-tasks \
--cluster myecscluster \
--tasks $AWS_ECS_TASK_ARN \
--query 'tasks[0].attachments[0].details[?name == `networkInterfaceId`].value' \
--output text) && 
AWS_ECS_APP_PUBLIC_IP=$(aws ec2 describe-network-interfaces \
--network-interface-ids $AWS_ECS_FARGATE_ENI \
--query 'NetworkInterfaces[0].Association.PublicIp' \
--output text) &&
echo $AWS_ECS_APP_PUBLIC_IP
```
- Open the public ip address (above output) in your browser Or curl the public ip address
```
curl $AWS_ECS_APP_PUBLIC_IP
```
