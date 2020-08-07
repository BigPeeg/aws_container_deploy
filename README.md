# AWS Container Deployment
Example setup of how to deploy an ICAP container to AWS ECS. 

## Prerequisities
Access credentials to an [AWS Account](https://aws.amazon.com/)
[AWS Command Line Interface](https://aws.amazon.com/cli/)
Docker runtime

The `eu-west-1` region will be used through this example

## Create the Image

Create the image  using `docker build`
```
docker build -t gw-icap:latest .
```

More detailed instructions are available in the [ICAP documentation](https://github.com/filetrust/c-icap/blob/master/Documentation/building_icap_docker_image.md).

## Push to Amazone ECR

The Elastic Container Registry (ECR) is a Docker container registry that is integrated with Amazon Elastic Container Service (ECS).

### Create the Image Registry

```
aws ecr create-repository --repository-name icap-pilot --region eu-west-1

```

On successful creation of the registry the details are reported in JSON format.
```
{
    "repository": {
        "repositoryArn": "arn:aws:ecr:eu-west-1:370377960141:repository/icap-pilot",
        "registryId": "370377960141",
        "repositoryName": "icap-pilot",
        "repositoryUri": "370377960141.dkr.ecr.eu-west-1.amazonaws.com/icap-pilot",
        "createdAt": "2020-07-08T11:39:59+01:00",
        "imageTagMutability": "MUTABLE",
        "imageScanningConfiguration": {
            "scanOnPush": false
        }
    }
}
```

### Push the Image

Using the `repositoryUri` in the JSON details, tag the local image
```
docker tag gw-icap:latest  370377960141.dkr.ecr.eu-west-1.amazonaws.com/icap-pilot:latest
```

Authenticate Docker to the ECR registry with `get-login-password`. When passing the authentication toke to the `docker login` command, use the value `AWS` for the username and specify the ECR URI.
```
aws ecr get-login-password --region eu-west-1 | docker login --username AWS --password-stdin 370377960141.dkr.ecr.eu-west-1.amazonaws.com/icap-pilot
```

Now push the local image to the ECR repository
```
docker push 370377960141.dkr.ecr.eu-west-1.amazonaws.com/icap-pilot:latest
```
Once the `push` is complete the image is available in the ECR

## Identify Execution Role
In order to support ECR images, the Fargate task definition requires a execution role. The `ecsTaskExecutionRole` is created automatically and its ARN can be used in the Fargate Task definition.

To find the ARN of the `ecsTaskExecutionRole`
1. Open the IAM console at https://console.aws.amazon.com/iam/
1. In the navigation pane, choose Roles.
1. Search the list of roles for `ecsTaskExecutionRole`. 
1. Click on the Role to open its summary page, the Role ARN is displayed.

## Create the Fargate task
Create a file call `icap-pilot-task.json` and populate it with the following configuration. The value of the `executionRoleArn` should the the Role ARN identified in the previous step.
```
{
    "family": "icap-pilot", 
    "networkMode": "awsvpc", 
    "executionRoleArn": "arn:aws:iam::370377960141:role/ecsTaskExecutionRole",
    "containerDefinitions": [
        {
            "name": "icap-pilot", 
            "image": "370377960141.dkr.ecr.eu-west-1.amazonaws.com/icap-pilot:latest", 
            "portMappings": [
                {
                    "containerPort": 1344, 
                    "hostPort": 1344, 
                    "protocol": "tcp"
                }
            ],            
            "essential": true
        }
    ], 
    "requiresCompatibilities": [
        "FARGATE"
    ], 
    "cpu": "256", 
    "memory": "2048"
}
```
Register the task with the following command
```
aws ecs register-task-definition --region eu-west-1 --cli-input-json file://icap-pilot-task.json
```

## Create the ECS Cluster
Create the cluster using the following command
```
aws ecs create-cluster --cluster-name "icap_pilot_cluster" --region eu-west-1
```

Before starting the Task we need to setup the networking.
The default security group in the VPC can be used, as the Fargate task only needs outbound connectivity.
```
aws ec2 describe-security-groups
```
The 'default' group's `GroupId` is returned in the JSON response.

Two public subnets are required. The available subnets can be listed using teh command
```
aws ec2 describe-subnets
```

Now launch the Fargate task in this cluster
```
aws ecs run-task --region eu-west-1 \
  --cluster "icap_pilot_cluster" \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-ea2c738c,subnet-3da98675],securityGroups=[sg-7ac07a30],assignPublicIp=ENABLED}" \
  --task-definition icap-pilot:1
  
  aws ecs run-task --region eu-west-1 --cluster "icap_pilot_cluster" --launch-type FARGATE --network-configuration "awsvpcConfiguration={subnets=[subnet-ea2c738c,subnet-3da98675],securityGroups=[sg-7ac07a30],assignPublicIp=ENABLED}" --task-definition icap-pilot:1
```

# References
[Environment variables to configure the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-envvars.html?icmpid=docs_sso_user_portal)
[Configuring the AWS CLI to use AWS Single Sign-On](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-sso.html)
[Securing credentials using AWS Secrets Manager with AWS Fargate](https://aws.amazon.com/blogs/compute/securing-credentials-using-aws-secrets-manager-with-aws-fargate/)
[Amazon ECR Registries : Registry Authentication](https://docs.aws.amazon.com/AmazonECR/latest/userguide/Registries.html)
[Amazon ECS Task Execution IAM Role](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_execution_IAM_role.html)