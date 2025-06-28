<pre>
Architecture
Image description

Prepare docker Image
Image description

Dockerfile - A Dockerfile is a text file that contains a series of instructions and commands used to build a Docker image. It specifies the operating system, application code, dependencies, environment variables, and other necessary configurations to create a Docker image.

Docker Image - A Docker image is a lightweight, standalone, executable package that includes everything needed to run a piece of software, including the code, runtime, libraries, environment variables, and configurations. It serves as a template for creating Docker containers. An image is built from a Dockerfile.

Docker Container - A Docker container is a runtime instance of a Docker image. It is a lightweight, standalone, and executable unit that runs the software defined in the Docker image. Containers are used to run applications in isolation from the host system and other containers.

Create a python script
Create file demp.py in a new directory LightningTalk and add code in it.

Create a docker file
Create a file docker file in same directory



touch Dockerfile


Add following code in Dockerfile



FROM public.ecr.aws/docker/library/python:3

WORKDIR /home/harsh/workplace/LightningTalk

COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD [ "python", "./demo.py" ]


We are using python base image from ECR Public Gallery https://gallery.ecr.aws/docker/library/python/?page=1

Build Docker Image


docker build --platform linux/amd64 -t lightning-talk-image:test .


Run container for testing


docker run -it --rm --name lightning-talk-task lightning-talk-image:test



Create ECR Registry and push image to ECR
Run the get-login-password command to authenticate the Docker CLI to your Amazon ECR registry.



aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin $AWSACCOUNTID.dkr.ecr.us-east-1.amazonaws.com



After you have authenticated to an Amazon ECR registry with this command, you can use the client to push and pull images from that registry

Create a repository in Amazon ECR using the create-repository command.



aws ecr create-repository --repository-name python-images --region us-east-1 --image-scanning-configuration scanOnPush=true --image-tag-mutability MUTABLE


Run the docker tag command to tag your local image into your Amazon ECR repository as the latest version. Copy the repositoryUri from the output in the previous step.



docker tag lightning-talk-image:test $AWSACCOUNTID.dkr.ecr.us-east-1.amazonaws.com/python-images:latest


Run the docker push command to deploy your local image to the Amazon ECR repository. Make sure to include :latest at the end of the repository URI.



docker push $AWSACCOUNTID.dkr.ecr.us-east-1.amazonaws.com/python-images:latest


Create ECS Resources and run the task
Create a new cluster


aws ecs create-cluster --cluster-name lightning-talk-cluster


Register a Task Definition
Before you can run a task on your ECS cluster, you must register a task definition. Task definitions are lists of containers grouped together.



aws ecs register-task-definition --cli-input-json file://./fargate-task.json


Before running above command we need to save the task definition JSON as a file fargate-task.json.



{
    "family": "python-tasks",
    "containerDefinitions": [
        {
            "name": "lightning-talk",
            "image": "$AWSACCOUNTID.dkr.ecr.us-east-1.amazonaws.com/python-images:latest",
            "cpu": 256,
            "memory": 2048,
            "portMappings": [
                {
                    "containerPort": 80,
                    "hostPort": 80,
                    "protocol": "tcp"
                }
            ],
            "essential": true,
            "environment": [],
            "environmentFiles": [],
            "mountPoints": [],
            "volumesFrom": [],
            "ulimits": [],
            "logConfiguration": {
                "logDriver": "awslogs",
                "options": {
                    "awslogs-group": "/ecs/python-tasks",
                    "awslogs-create-group": "true",
                    "awslogs-region": "us-east-1",
                    "awslogs-stream-prefix": "ecs"
                },
                "secretOptions": []
            },
            "systemControls": []
        }
    ],
    "executionRoleArn": "arn:aws:iam::$AWSACCOUNTID:role/ecsTaskExecutionRole",
    "networkMode": "awsvpc",
    "requiresCompatibilities": [
        "FARGATE"
    ],
    "cpu": "256",
    "memory": "2048"
}


Running a task
Run following task



aws ecs run-task --cluster lightning-talk-cluster --task-definition python-tasks --launch-type FARGATE --network-configuration 'awsvpcConfiguration={subnets=["subnet-xxxxx","subnet-xxxxx"],securityGroups=["sg-xxxxxx"],assignPublicIp="ENABLED"}'


To get the values of subnet and subnet we can use values from default VPC and default security group.
</pre>
