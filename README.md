# audio-feature-extraction
A lambda function for audio feature extraction

Requirements:
* AWS account
* Docker installed locally - (https://www.docker.com/products/docker-desktop)
* AWS Command Line Interface installed locally - (https://aws.amazon.com/cli/)
* ECR Repository
* Sonic Annotator library(linux) - (https://code.soundsoftware.ac.uk/projects/sonic-annotator/files)
* pYIN plugin(linux) - (https://code.soundsoftware.ac.uk/projects/pyin/files)

We decided to use Docker to containerize and package the Lambda function locally since the Sonic Annotator tool requires Linux libraries that are not provided by default in the Lambda function environment. Starting with the Dockerfile, we declare the Lambda function manifest, beginning with which base image it’s going to use.

`FROM amazon/aws-lambda-python:3.8`


This specifies that our application will use Python 3.8, one of AWS’s default images that also supports the Lambda function environment.

`COPY app.py ./` 

Here we copied the Python script which contains the Lambda handler to the root directory of the Docker image.
The bin directory contains sonic-annotator.so and also the Vamp directory, which itself contains the pYIN plugins. The .so files corresponding to the sonic-annotator and pYIN plugins can be extracted from the sonic-annotator and pYIN downloaded files.

/bin

├── vamp

│   └── pyin.so

└── sonic-annotator.so

We unpack the files in the bin directory to the root directory of the Docker image as well by copying the files that are inside the bin directory at the working directory.

`RUN yum -y install qt`
`RUN yum -y install qt5-qtbase`
`RUN yum -y install qt5-qtbase-common`

These three yum commands are run while the Docker image is being built and installs the needed Linux dependencies that allows sonic-annotator to run.

`ENV VAMP_PATH $pwd"vamp"`

The VAMP_PATH variable is used by the sonic-annotator tool to know where the plugins are located, and it’s set as an environment variable which is equal to the root path plus the vamp directory.


`CMD ["app.handler"]`

This part specifies the commands that the Docker container should execute when it starts to run i.e., the function handler inside the app.py script.



## Building the Docker image


To build the Docker image locally, make sure that you’re inside the application directory where the Dockerfile is located and run:

`docker build -t <image-name>:<tag> . `

This command will package the application into a Docker image with its required dependencies, the name sonic and the tag latest.


## Deploying the Docker image to AWS


At this stage of the deployment, we’re going to use the ECR service which allows us to store our Docker images. First of all, you need to login to this service locally by running the following command in the AWS CLI.

`aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <account_id>.dkr.ecr.<region>.amazonaws.com`

After logging in, begin by tagging the local Docker image before pushing it to the ECR repository.

`docker tag <image:tag> <account_id>.dkr.ecr.<region>.amazonaws.com/<image:tag>`

After the image is properly tagged, you can run the docker push command which will push the tagged image into the ECR repository.

`docker push <account_id>.dkr.ecr.<region>.amazonaws.com/<image:tag>`


If you’re trying to push the image into a repository which doesn’t exist, the command will automatically create a new repository with the name of the image that is being pushed there.



## Create the AWS Lambda function

Go the AWS web console, navigate to the Lambda service and create a new function. After giving the basing information for the Lambda function and assigning the Lambda role that has access to the S3 write and read to the S3 buckets, choose the Container Image option to deploy the container image:

After creating the Lambda function with the Container Image deployment option, open up the Lambda function and choose to deploy new image:



After choosing to Deploy a new image, you will be directed to choose the image that you want to deploy into the Lambda function.



By browsing container images, you can see the existing repositories and also the images that are stored inside those repositories. Make sure to select the latest one.


 After selecting the image, it will be deployed to the Lambda function and every time the Lambda function gets invoked, a container from that image is created and will be executed.


## Lambda function documentation

After the Lambda function is created with the container image, it will be triggered when a file is uploaded to the S3 bucket and we’ll fetch the name of the file from the event that the function receives.
The file name will be parsed if it gets quoted in the event, which happens if the uploaded file contains spaces in its name for example:
file name.csv -> file+name.csv (“ “ gets replaced with “+”)
filename (1).csv -> filename2314.csv (“ (1) ” gets replaced with “2314”)
After we receive the file name parsed correctly, we download the file from the S3 bucket into the /tmp/ directory inside the Lambda function container since you’re not allowed to store files.
After the file is downloaded, we give the path where the file is located to the sonic-annotator command and replace any character from the file name that the command line wouldn’t accept.
The --csv-stdout parameter is added to the command so that we don’t store any file locally but rather store the output in memory as a variable, then store that output in the S3 bucket.




