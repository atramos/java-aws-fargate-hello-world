# Welcome
This is a very simple java project that will deploy a "Hello World" Java Webapp into AWS Fargate using only the AWS Java APIs and the java-docker library wrapper for Docker.  This is convenient for defining your deployment in code.

# Roles

The roleARN is the role that ECS will assume to be able to start EC2 instances for you.  It needs to have the following:

1. Attach an inline policy to allow manipulation of containers:

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ecr:CreateRepository",
                "ecr:GetAuthorizationToken"
            ],
            "Resource": "*"
        }
    ]
}

Note, the AWS managed Policy AmazonEC2ContainerServiceforEC2Role is insufficient as it lacks ecr:CreateRepository.


1. Add a Trust Relationship to ecs-tasks.amazonaws.com:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "Service": "ecs-tasks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

The deploying user needs the following role so that the deploying user can pass the above role on to ECS (feel free to restrict the resource to just the roleArn rather than * for more security, otherwise the deploying user has root):
```
"Effect": "Allow",
"Action": [
    "iam:GetRole",
    "iam:PassRole"
],
"Resource": "*"
```


Without the above three things, you'll get errors when you run `aws ecs describe-services --cluster ad-systems --services fargate-demo` leads to error `ECS was unable to assume the role`.

# TL;DR

The quickest way to get started is to manually create a Role in IAM through the AWS Console, add the above Policies and Trust Relationship to the Role, then launch an Ubuntu t2.nano EC2 instance using the Role as IAM role, and execute the following commands inside the EC2 instance:

```
apt-get install docker jq aws-cli git maven
git clone https://github.com/ratamacue/java-aws-fargate-hello-world
cd java-aws-fargate-hello-world
mvn install
INTERFACE=$(curl --silent http://169.254.169.254/latest/meta-data/network/interfaces/macs/)
SUBNET_ID=$(curl --silent http://169.254.169.254/latest/meta-data/network/interfaces/macs/${INTERFACE}/subnet-id)
ROLE_ARN=$(curl --silent http://169.254.169.254/latest/meta-data/iam/info|jq -r .InstanceProfileArn)
./deploy.sh -rolearn $ROLE_ARN -appname hello -subnets $SUBNET_ID -awsid ${ROLE_ARN:13:12} -dockerport 8080 -dockertag hello -clustername hello -cpu 1 -memory 2G
```



