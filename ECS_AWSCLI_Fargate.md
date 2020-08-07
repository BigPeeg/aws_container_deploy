# Creating a Cluster with a Fargate Task Using the AWS CLI

## Create a Cluster

```
aws ecs create-cluster --cluster-name icap-pilot-cluster
```

## Register a Task Definition

```
aws ecs register-task-definition --cli-input-json file://icap-pilot-task.json
```

## Create a service using a public subnet
```
aws ecs create-service --cluster icap-pilot-cluster  --service-name icap-pilot-service --task-definition icap-pilot:2 --desired-count 1 --launch-type "FARGATE" --network-configuration "awsvpcConfiguration={subnets=[subnet-3da98675,subnet-e660f9bc],securityGroups=[sg-0f07a8bc71b2cc92c],assignPublicIp=ENABLED}"
```





# References
https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_AWSCLI_Fargate.html