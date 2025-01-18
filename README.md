# Notes on provisioning the ECS ExecuteCommand for terminal access to ECS Fargate containers 

The solution below has a lot of prerequistes but works once aws cli2 and SSM are installed.
I performed the terminal access from the Mac

Will clean this note up when have the time.

## Basic steps: Role provisioning

This is the Task Role not the Task Execution Role(remove the policy above from Task Execution Role)

https://stackoverflow.com/questions/71244323/iam-role-to-allow-command-execution-on-aws-ecs-containers


In your resource "aws_ecs_task_definition" "app" you have specified an execution_role_arn, but you have not specified a task_role_arn. That's really what the error is saying, that you need to provide a task role ARN.
The execution role gives the ECS service permission to do things like read an image from an ECR repository, and lookup secrets in SecretsManager that it needs to inject into the containers it creates.
The task role gives the software running inside the ECS task/container permission to access AWS resources. The command execution permissions need to be assigned to the task role, not the execution role.
At a minimum you could try adding:
task_role_arn = aws_iam_role.ecs_task_execution_role.arn
But following the principal of least privilege would dictate you separate those roles into separate IAM roles with distinct privileges



https://github.com/docker-archive/compose-cli/issues/2120



This is not an issue with compose-cli. You need to provide a "Task role" for a Task Definition (this is different than the "Task execution role"). This can be done by first going to IAM
IAM role creation
1.	IAM > roles > create role
2.	custom trust policy > copy + paste
{
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
}
3.	Add permission > Create Policy
4.	JSON > replace YOUR_REGION_HERE & YOUR_ACCOUNT_ID_HERE & CLUSTER_NAME > copy + paste
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ssmmessages:CreateControlChannel",
                "ssmmessages:CreateDataChannel",
                "ssmmessages:OpenControlChannel",
                "ssmmessages:OpenDataChannel"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "logs:DescribeLogGroups"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogStream",
                "logs:DescribeLogStreams",
                "logs:PutLogEvents"
            ],
            "Resource": "arn:aws:logs:YOUR_REGION_HERE:YOUR_ACCOUNT_ID_HERE:log-group:/aws/ecs/CLUSTER_NAME:*"
        }
    ]
}
5.	Give it a name
6.	go back to Add permissions > search by name > check > Next
7.	Give a role name > create role



Unable to create new policy. Add the role an then go back and create the policy and add the policy to the role



## revise the task version


>>>>Create this new role and bind to the task revision

The role is ecsTaskRole and not ecsTaskExecutionRole !!!!!!!!!!!!


ECS new task
1.	go back to ECS > go to task definition and create a new revision
2.	select your new role for "Task role" (different than "Task execution role") > update Task definition
3.	go to your service > update > ensure revision is set to latest > finish update of the service
4.	current task and it should auto provision your new task with its new role.
5.	try again


What i had to do is repush from VSCode to github so that workflow5 in github actions would redeploy. This installed the new latest task version into the currently running service that is bound to the ECS cluster



The updated task definition JSON file should have the following (both are required and the taskRoleArn was newly added as indicated above. This is the one that provisions the ExecuteCommand access to the running container in ECS)

    "taskRoleArn": "arn:aws:iam::72529XXXXXXX:role/ecsTaskRole",
    "executionRoleArn": "arn:aws:iam::72529XXXXXX:role/ecsTaskExecutionRole",


## The taskRoleArn is now present but the service is still running old task revision from 4 horus ago.


Try pushing a new change so that the new task revision is used and re-deployed to the service


First update the task definition in source code in aws-files directory. This now has the taskRoleArn above so that we can force the ExecuteCommand functionality on the service (see below)


## Configuration command and verification commands


Try running the command again now that the service is running with the updated task revision

bash-5.2$ aws ecs update-service --cluster github-actions-ecs-vprofile-project12 --service github-actions-ecs-service-vprofile-project12-2 --force-new-deployment --enable-execute-command

This enables the execute command
This will be at the end of the output:         "enableExecuteCommand": true


### This can be verified with the following command:

>>>>aws ecs describe-tasks \
    --cluster github-actions-ecs-vprofile-project12 \
    --region $AWS_REGION \
    --tasks a01948d263a44a828fe03xxxxxxxxxxxx   <<<< latest task id after repushed code and started a new service instance with new revision task and forced a new deployment (with same ECR image)


This should be present in the output:
                    "managedAgents": [
                        {
                            "lastStartedAt": "2025-01-17T17:39:49.966000-08:00",
                            "name": "ExecuteCommandAgent",
                            "lastStatus": "RUNNING"
                        }
                    ],

And this:
"enableExecuteCommand": true,


## finally terminal in with the aws ecs execute-command:

aws ecs execute-command --region REGION --cluster CLUSTER_NAME --task $TASK_ARN --container CONTAINER --command "sh" --interactive

aws ecs execute-command  \
    --region $AWS_REGION \
    --cluster github-actions-ecs-vprofile-project12 \
    --task c40bbf75faea4c08aeae68xxxxxxx \
    --container vprofile-project12 \
    --command "/bin/bash" \
    --interactive


    aws ecs execute-command --region us-east-1 --cluster github-actions-ecs-vprofile-project12 --task a01948d263a44a82xxxxxxxx --container vprofile-project12 --command "sh" --interactive



bash-5.2$ aws ecs execute-command --region us-east-1 --cluster github-actions-ecs-vprofile-project12 --task a01948d263a44a828fexxxxxxxx --container vprofile-project12 --command "sh" --interactive

The Session Manager plugin was installed successfully. Use the AWS CLI to start a session.


Starting session with SessionId: ecs-execute-command-q3xq89rrf76q9x22ycfxxxxxxxx


## test out a few re-deployments and make sure that can still terminal in (the task id will change each time with each new task revision that is running in the ECS service in the cluster):

working fine on redeployment

bash-5.2$ aws ecs execute-command --region us-east-1 --cluster github-actions-ecs-vprofile-project12 --task arn:aws:ecs:us-east-1:725291656587:task/github-actions-ecs-vprofile-project12/95854dfe6c13481eaae8417xxxxxxx--container vprofile-project12 
--command "sh" --interactive

The Session Manager plugin was installed successfully. Use the AWS CLI to start a session.


Starting session with SessionId: ecs-execute-command-hyngbc4afbsctgvxxxxxxxx



# github actions workflows final workflow stage 5 with all the stages:

This file is under the following directory in the source code and is working very well.

davemastropolo@Margarets-MacBook-Air workflows % pwd
/Users/davemastropolo/workspace/prod/course8_devops_zproject12_Github_Actions_CICD/hprofile/.github/workflows
davemastropolo@Margarets-MacBook-Air workflows % ls -la
total 72
drwxr-xr-x  8 davemastropolo  staff    256 Jan 17 18:26 .
drwxr-xr-x  3 davemastropolo  staff     96 Mar 25  2024 ..
-rw-r--r--  1 davemastropolo  staff    418 Jan 16 10:50 main.yml
-rw-r--r--  1 davemastropolo  staff    831 Mar 25  2024 main_copy.yml
-rw-r--r--  1 davemastropolo  staff  10175 Jan 17 18:26 main_copy_sonar_build_publish_deploy_stage5.yml
-rw-r--r--  1 davemastropolo  staff   6538 Jan 16 19:21 main_copy_sonar_build_publish_stage4.yml
-rw-r--r--  1 davemastropolo  staff   3225 Jan 15 18:08 main_copy_sonar_stage2.yml
-rw-r--r--  1 davemastropolo  staff   3759 Jan 15 18:45 main_copy_sonar_stage3.yml



# Next port this to the VPS gitlab container so that push from VSCode to gitlab container on VPS is then pushed to github to run these scripts. The VPS has all of the latest development scripts (Jenkins CI/CD, native gitlab with kops deployment, github actions and AWS CodePipeline via a connector configured on gitlab):

WIP





# Prerequisites
# test commit to the forked dmastrop/hprofile repository
#####
# test commit with workflow on push
- JDK 11
- Maven 3
- MySQL 8 

# Technologies 
- Spring MVC
- Spring Security
- Spring Data JPA
- Maven
- JSP
- MySQL
# Database
Here,we used Mysql DB 
MSQL DB Installation Steps for Linux ubuntu 14.04:
- $ sudo apt-get update
- $ sudo apt-get install mysql-server

Then look for the file :
- /src/main/resources/db_backup.sql
- db_backup.sql file is a mysql dump file.we have to import this dump to mysql db server
- > mysql -u <user_name> -p accounts < db_backup.sql
