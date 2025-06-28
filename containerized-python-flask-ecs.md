<pre>
Table of Contents
Overview
Design Choices
Architecture
Terraform Architecture
Setup and Installation
Usage
Testing
Future Enhancements
Overview
The Echo Service is a Flask application designed to allow clients to create mock endpoints with pre-defined responses. It offers flexibility in creating, modifying, and deleting these mock endpoints.You can find the code here.

Design Choices
Framework : I have used the Flask framework due to its simplicity and because it’s a lightweight framework.
Aws Architecture : For deploying Flask application on AWS , we choose ECS with AWS Fargate.
Data Classes: To ensure type safety and better documentation, Python data classes were used to define the data models.
Error Handling: Custom error handlers were implemented to provide detailed and user-friendly error messages.
Logging: Standard Python logging was implemented for easy troubleshooting and understanding the system flow.
Infrastructure-as-Code (IaC): The application is designed to be deployed on AWS using Terraform.
Containerization : Docker ensures that the application runs uniformly across different environments by packaging the app and its dependencies together
AWS Architecture Choices
I had three main options for deploying my flask application on Aws:
1)AWS Lambda with API Gateway — If you want to minimize maintenance and your traffic is sporadic, AWS Lambda is compelling for its simplicity
2)ECS with AWS Fargate — If you need fine-grained control, predictable scaling and want predictable costs, ECS with Fargate is a strong contender
3)AWS Elastic Beanstalk — For generalized solutions or when starting out, Elastic Beanstalk is a balanced choice
Architecture
The application follows the MVC pattern with a slight deviation given Flask’s architectural preferences.
Model: Defined using Python data classes in models.py.
View: All endpoint functions are defined in views.py.
Controller: While Flask merges views and controllers, additional business logic can be added in separate controllers/services if needed.
Error handling is modularized in errors.py.
The application is designed to be stateless, allowing it to scale horizontally if deployed in a cloud environment.
Terraform Architecture
Using Infrastructure as Code (IaC) with Terraform provides a flexible and repeatable deployment process. This application is designed to be deployed on AWS, leveraging various services.

AWS Resources
Virtual Private Cloud (VPC): Provides a secure and isolated environment for resources.
High Availability with Subnets: By deploying the application across multiple subnets in different availability zones, we ensure that the application remains available even if one of the AWS data centers fails.
Internet Gateway: To allow the Flask application to communicate with the external world, for example, to fetch updates or external APIs.
ECS with Fargate: Used to run the Flask application in containers, without managing underlying infrastructure.
Elastic Load Balancer (ELB): Distributes incoming application traffic across multiple targets.
IAM Roles and Policies: Gives permissions to AWS services to operate on your behalf.
Security Groups: Acts as a virtual firewall, controlling incoming and outgoing traffic.
Auto-Scaling : Automatically adjusts the number of ECS tasks based on defined metrics, ensuring application responsiveness during traffic surges and efficient resource usage during lulls.
CloudWatch Logging and Dashboard: Monitors application performance and health, aggregating logs and metrics to offer insights and facilitate troubleshooting
Setup and Installation
clone the code from the repo.
Navigate to the project folder.
Install the required packages: pip install -r requirements.txt.
Run the application locally: python main.py.
For Docker setup:
Build the Docker image: docker build -t echo-service ..
Run the Docker container: docker run -p 5000:5000 echo-service.
6. Ensure that you build the Docker image and push it to a container registry (in our case its AWS Elastic Container Registry).

7. The var.docker_image_url in your Terraform code should then be set to the URL of this pushed image

# --- Build Stage ---
FROM python:3.8-slim as builder

WORKDIR /app

# Install build dependencies
COPY requirements.txt /app/
RUN pip install --upgrade pip \
    && pip install --user --no-cache-dir -r requirements.txt

# Let's verify what's installed
RUN pip list --user

# Let's inspect the directory structure to make sure we're copying the right directories later
RUN ls -la /root/.local/bin

# --- Production Stage ---
FROM python:3.8-slim as production

RUN pip install gunicorn

# Set the working directory and create app user
WORKDIR /app
RUN useradd --create-home appuser && chown -R appuser:appuser /app
USER appuser

# Copy only the necessary files
# Copy Python dependencies installed in the builder stage
COPY --from=builder /root/.local /home/appuser/.local
# Copy application files
COPY . /app

# Ensure scripts in .local are usable:
ENV PATH=/home/appuser/.local/bin:$PATH

ENV PYTHONPATH=/home/appuser/.local/lib/python3.8/site-packages:$PYTHONPATH

RUN echo $PYTHONPATH


RUN ls -la /home/appuser/.local/bin

# Environment variables for Flask
ENV FLASK_ENV=production
ENV FLASK_DEBUG=0

# Expose port and set CMD
EXPOSE 5000

CMD ["/home/appuser/.local/bin/gunicorn", "-b", "0.0.0.0:5000", "main:app"]
Deployment
Ensure AWS CLI is set up and credentials are configured.
Initialize Terraform: terraform init
Apply the configuration: terraform apply
Testing the Application
Once you’ve deployed your Flask application using Terraform, accessing it via a browser involves a few steps. Based on your Terraform script, you’re using AWS ECS with Fargate launch type and an Application Load Balancer (ALB) to route the traffic to your Flask application.

Here’s how you can access your Flask application in a browser:

Get the Load Balancer URL:
After you’ve applied your Terraform configuration, AWS will provision an Application Load Balancer with a DNS name. You can obtain this DNS name from the AWS Management Console or using the AWS CLI.

Via AWS Console:
Navigate to the EC2 Dashboard.
In the navigation pane, under “Load Balancing”, click on “Load Balancers”.
Select your Load Balancer (named “flask-lb” based on your Terraform code).
In the description tab, copy the “DNS name” (It should look something like flask-lb-1234567890.us-west-1.elb.amazonaws.com).
Via AWS CLI:
aws elbv2 describe-load-balancers --names "flask-lb" --query "LoadBalancers[0].DNSName"
2. Access the Application:

Paste the Load Balancer DNS name into your browser’s address bar, and it should route the traffic to one of the running Flask tasks and display your application. For example:

http://flask-lb-1234567890.us-west-1.elb.amazonaws.com
3. Domain Name (Optional):

If you have a custom domain name and you’d like to use that instead of the load balancer’s DNS name:

Purchase a domain from a registrar (or use an existing one).
Use Amazon Route 53 or another DNS service to create an A record (or an ALIAS record if using Route 53) pointing to the Load Balancer's DNS name.
Once DNS propagation is complete (which can take anywhere from a few minutes to 48 hours), you can access your Flask app using your custom domain name.
4.Secure with HTTPS (Optional):

For production environments, it’s recommended to serve applications over HTTPS:

Obtain an SSL/TLS certificate for your domain. You can get a free certificate from Let’s Encrypt or purchase one from commercial providers.
Use AWS Certificate Manager (ACM) to provision the certificate in AWS.
Update the ALB settings (in Terraform or AWS Console) to use the certificate and listen on HTTPS (port 443) instead of HTTP.
Update security group rules to allow traffic on port 443.
Access your application via https://yourdomain.com.
Remember that it might take a few minutes for the ECS tasks to start and the application to be accessible after deploying. Also, ensure that the Flask app inside the Docker container is set to listen on 0.0.0.0 and not 127.0.0.1, as the latter will not accept external connections.

Cleanup
To prevent unnecessary costs, destroy the resources when done: terraform destroy

Manually creating AWS ECS With Fargate without a Terraform.
Step 1: Create an ECS Cluster
Navigate to the ECS service in the AWS Management Console.
Click on “Create Cluster” and configure cluster settings, including name, VPC, and subnet.
Choose Fargate as the launch type and click “Create”.
Step 2: Create an ECR Repository
Go to the ECR service and click on “Create Repository”.
Keep the repository public for easy access and provide a suitable name.
Click “Create Repository” to finish.
Step 3: Push Flask Application Image to ECR Repository
Setup Docker on your EC2 instance:
Clone the Flask app code from the GitHub repository
Build the Flask app image using Dockerfile
Before you can push your Docker image to your ECR repository, you need to authenticate your Docker CLI to the Amazon ECR registry to which you intend to push your image.
Use the aws ecr get-login-password command to get the authentication token.
Tag the container.
docker tag <repository-name>:latest <account-id>.dkr.ecr.<region>.amazonaws.com/<repository-name>:latest
7. Push the container to ECR Repository.

docker push <account-id>.dkr.ecr.<region>.amazonaws.com/<repository-name>:latest
Step 4: Create a Task Definition
In ECS service, click on “Task Definitions” and then “Create new Task Definition”.
Set the container name to “run-flask-app” and paste the Image URL from ECR.
Specify port mappings for HTTP (Container port: 80).
Choose Fargate launch type and configure task settings, including memory and CPU limits.
Review the configuration and create the task definition.
Step 5: Run the Task Definition
From the ECS cluster, click on “Deploy” and then “Run Task”.
Select the cluster where the task should run.
Provide deployment configuration details, including VPC and security groups.
Launch the task and wait for it to be up and running.
Step 6: Access the Flask Application
Once the task is running, browse the Public IP address to access the Flask application.
You should now see the default Flask app running successfully.
Usage
Detailed API documentation is provided in the api_docs.md file. The primary endpoints are:

POST /endpoints: Create a new mock endpoint.
GET /endpoints: List all the created mock endpoints.
PATCH /endpoints/:id: Update an existing mock endpoint.
DELETE /endpoints/:id: Delete an existing mock endpoint.
The created mock endpoints can be accessed via their defined paths and HTTP methods.
Testing
Tests are written using pytest.

Navigate to the root project folder.
Run the tests: pytest.
Future Enhancements
Database Integration: Integrate a database to persistently store the mock endpoints.
Authentication & Authorization: Integrate a complete Auth system.
Multi-Stage-Dockerfile: Multi-stage builds are used in Docker to minimize the final image size, typically by separating the build environment from the run environment. This is especially useful when building binaries from source code, but can also help in Python applications by excluding build tools and other unnecessary files. For a Flask application, the benefit of multi-stage builds can be minimal since Python is an interpreted language and doesn’t generate separate binaries, but it still can be useful in specific contexts. Implement rate limiting to prevent abuse of the service.
Queue Service (RabbitMQ): Implement a messaging system like RabbitMQ to handle asynchronous tasks, ensuring the application remains responsive even under high loads.
Caching Mechanism: Integrate caching solutions like Redis or Memcached to boost performance and reduce database load.
Background Workers: Use tools like Celery with RabbitMQ as the message broker to process background jobs, improving response times and offloading non-immediate processing tasks.


</pre>
