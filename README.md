# aws-ecs

Reference to deploy to AWS ECS

## AWS CLI configuration

Configure

```zsh
aws configure --profile my-profile
```

If already created, get the existing profile to set it next

```zsh
aws configure list-profiles
```

Set for the current shell

```zsh
export AWS_PROFILE=my-profile
```

## Docker Build

Build

```zsh
docker build -t hello-world .
```

## Create ECR repository

```zsh
aws ecr create-repository --repository-name hello-world
```

```zsh
aws ecr get-login-password --region <your-region> | docker login --username AWS --password-stdin <your-account-id>.dkr.ecr.<your-region>.amazonaws.com
```

## Docker Tag and Push

```zsh
docker tag hello-world:latest <your-account-id>.dkr.ecr.<your-region>.amazonaws.com/hello-world:latest
docker push <your-account-id>.dkr.ecr.<your-region>.amazonaws.com/hello-world:latest
```

## Task Creation

Create a file named `task-def.json`

```zsh
{
  "family": "hello-world-task",
  "networkMode": "awsvpc",
  "containerDefinitions": [
    {
      "name": "hello-world",
      "image": "<account-id>.dkr.ecr.<region>.amazonaws.com/hello-world:latest",
      "portMappings": [
        {
          "containerPort": 3000,
          "protocol": "tcp"
        }
      ],
      "essential": true
    }
  ],
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::<account-id>:role/ecsTaskExecutionRole"
}
```

## Create the Role

Create the `ecsTaskExecutionRole` role
```zsh
aws iam create-role --role-name ecsTaskExecutionRole --assume-role-policy-document file://<(echo '{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "ecs-tasks.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}')
```

Attach it 
```zsh
aws iam attach-role-policy --role-name ecsTaskExecutionRole --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
```

## Create a service

Find and pick a subnet

```zsh
aws ec2 describe-subnets --query "Subnets[*].[SubnetId,VpcId,MapPublicIpOnLaunch]"
```

Find and pick a security group:

```zsh
aws ec2 describe-security-groups --query "SecurityGroups[*].{ID:GroupId,Name:GroupName}" 
```

Make sure the security group selected has the following inbound rules, if not set it:
* Type: Custom TCP
* Protocol: TCP
* Port Range: 3000
* Source: 0.0.0.0/0 (for open access; restrict this later for security).


```zsh
aws ecs create-service \
  --cluster hello-world \
  --service-name hello-world-service \
  --task-definition hello-world-task \
  --desired-count 1 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[<subnet-id-1>,<subnet-id-2>],securityGroups=[<security-group-id>],assignPublicIp=ENABLED}"
```

## Verify the app is running use the Public IP

Navigate to the cluster, select the Task, and find the public IP. Verify it's running:

```zsh
http://<public-ip>:3000
```

## Create the Application Load Balancer

Create the ALB

```zsh
aws elbv2 create-load-balancer \
  --name hello-world-alb \
  --type application \
  --scheme internet-facing \
  --subnets <subnet-1-id> <subnet-2-id> \
  --security-groups <security-group-id> \
  --ip-address-type ipv4 
```

Create a target group

```zsh
aws elbv2 create-target-group \
  --name hello-world-target-group \
  --protocol HTTP \
  --port 3000 \
  --vpc-id <vpc-id> \
  --health-check-protocol HTTP \
  --health-check-path / \
  --target-type ip
```

Create a listener:

```zsh
aws elbv2 create-listener \
  --load-balancer-arn <alb-arn> \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=<target-group-arn>
```

Update the service to use the ALB:

```zsh
aws ecs update-service \
  --cluster <your-cluster-name> \
  --service <your-service-name> \
  --load-balancers "targetGroupArn=<target-group-arn>,containerName=<container-name>,containerPort=3000" \
  --force-new-deployment
```