# aws-cli-lambda
A script to package AWS CLI as a Lambda Layer. Copying the article written by the author at: https://bezdelev.com/hacking/aws-cli-inside-lambda-layer-aws-s3-sync/
into a quick Readme for better collating the code with article , later when required.

Ilya Bezdelev
How to use AWS CLI within a Lambda function (aws s3 sync from Lambda)
January 19, 2019 · 6 min read

# A step-by-step process to enable AWS CLI within an AWS Lambda function.

In general, when you want to use AWS CLI in Lambda, it's best to call AWS APIs directly by using the appropriate SDK from your function's code. But there's one specific use case that is easier to do with the AWS CLI than API: aws s3 sync. This post will show how to enable AWS CLI in the Lambda execution environment.

Motivation
If you want to copy files from S3 to the Lambda environment, you'd need to recursively traverse the bucket, create directories, and download files. This is not fun to build and debug.

Instead, the same procedure can be accomplished with a single-line AWS CLI command s3 sync that syncs the folder to a local file system. It also allows additional flags like --exclude to restrict what gets synced.

However, the Lambda execution environment doesn't have the AWS CLI pre-installed and neither can you install it using pip.

Solution
To get the CLI into Lambda, we will create a Lambda Layer that contains all the necessary dependencies and make it available to the function.

Install AWS CLI in a local virtual environment
Package AWS CLI and all its dependencies to a zip file
Create a Lambda Layer
Test if it works
Step-by-step guide
Note: These instructions were created for MacOS and Linux. If you run Windows, instructions will still work but you might need to replace bash variables with hardcoded directory names.

AWS CLI is a collection of Python scripts that use the boto3 library. For this guide to work, you'll need Python3 and pip installed on your local machine. You will also need virtualenv. Refer to the installation guide if you don't have virtualenv installed.

You can verify if you already have these tools by checking their version from the terminal.

```
$ python --version
Python 3.7.0

$ pip --version
pip 18.0 from /Library/Frameworks/Python.framework/Versions/3.7/lib/python3.7/site-packages/pip (python 3.7)

$ virtualenv --version
16.2.0
If you value efficiency, steps 1 through 7 are also available as a shell script in my github repo aws-cli-lambda.

git clone https://github.com/ilyabezdelev/aws-cli-lambda.git
```

Let's go

1. Prepare a directory & environment variables
Open the terminal, create a working directory for this exercise and cd into it.
Run the following commands in the terminal.
# Automatically detects python version (only works for python3.x)
export PYTHON_VERSION=`python3 -c 'import sys; version=sys.version_info[:3]; print("{0}.{1}".format(*version))'`

```
# Temporary directory for the virtual environment
export VIRTUAL_ENV_DIR="awscli-virtualenv"

# Temporary directory for AWS CLI and its dependencies
export LAMBDA_LAYER_DIR="awscli-lambda-layer"

# The zip file that will contain the layer
export ZIP_FILE_NAME="awscli-lambda-layer.zip"
```

2. Create a virtual environment
To get all AWS CLI dependencies conveniently stored in a separate directory, we'll create an isolated virtual environment using virtualenv and activate it.

```
# Creates a directory for virtual environment
mkdir ${VIRTUAL_ENV_DIR}

# Initializes a virtual environment in the virtual environment directory
virtualenv ${VIRTUAL_ENV_DIR}

# Changes current dir to the virtual env directory
cd ${VIRTUAL_ENV_DIR}/bin/

# Activate virtual environment
source activate
Your terminal prompt will now indicate that you're in a virtual envioronment. The prompt should be prepended with (awscli-virtualenv) like this:

(awscli-virtualenv) mymac:bin username$
```

3. Install AWS CLI in the virtual environment
The following command will install AWS CLI and all its dependencies to the virtual environment.

```
pip install awscli
```

4. Change the path to python
The AWS CLI executable is the file named aws and its first line provides the path to the Python interpreter. Since we're using a virtual environment, it will have a path local to the environment we created: #!/PATH_TO_WORKING_DIR/awscli-virtualenv/bin/python3.7.

We need to replace this path with the path to the Python interpreter in the Lambda execution environment, which is #!/var/lang/bin/python.

To replace the path to Python in the aws executable, you can run this command on MacOS.

```
sed -i '' "1s/.*/\#\!\/var\/lang\/bin\/python/" aws
… or this command on Linux, which has slightly different syntax because the implementation of sed on MacOS differs from that on Linux.

sed -i "1s/.*/\#\!\/var\/lang\/bin\/python/" aws
```

Alternatively, you can open aws in a text editor and replace the first line by hand.

5. Deactive virtualenv
Now we'll deactivate the virtual environment.

```
deactivate
```

The terminal prompt should have switched back to its normal state (no more (awscli-virtualenv) in the prompt).

6. Package AWS CLI and its depenedencies in a zip file
The following set of commands will prepare a temporary directory for packaging AWS CLI, copy all dependencies, and zip them.

```
# Changes current directory back to where it started
cd ../..

# Creates a temporary directory to store AWS CLI and its dependencies
mkdir ${LAMBDA_LAYER_DIR}

# Changes the current directory into the temporary directory
cd ${LAMBDA_LAYER_DIR}

# Copies aws and its dependencies to the temp directory
cp ../${VIRTUAL_ENV_DIR}/bin/aws .
cp -r ../${VIRTUAL_ENV_DIR}/lib/python${PYTHON_VERSION}/site-packages/ .

# Zips the contents of the temporary directory
zip -r ../${ZIP_FILE_NAME} *
```

7. Clean up
Remove awscli-virtualenv and awscli-lambda-layer directories manually or with the following commands.

```
# Goes back to where it started
cd ..

# Removes virtual env and temp directories
rm -r ${VIRTUAL_ENV_DIR}
rm -r ${LAMBDA_LAYER_DIR}
```

8. Test
Go to the Lambda console, click on Layers in the left navigation and follow the wizard to create a layer. Upload awscli-lambda-layer.zip.
Create a Lambda function using the same version of Python that was used for packaging AWS CLI. Set timeout to 15 seconds and memory limit to 512 MB (I found AWS CLI to be a little too slow in functions with less than 512 MB of memory).
After the function is created, in Designer, click on Layers, click Add layer and select the layer you've just created.
Paste the following code and execute the function by clicking the Test button (create a test event from the HelloWorld template when asked). The function should run in under 5 seconds and return AWS CLI's version in logs.
```
import subprocess
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def run_command(command):
    command_list = command.split(' ')

    try:
        logger.info("Running shell command: \"{}\"".format(command))
        result = subprocess.run(command_list, stdout=subprocess.PIPE);
        logger.info("Command output:\n---\n{}\n---".format(result.stdout.decode('UTF-8')))
    except Exception as e:
        logger.error("Exception: {}".format(e))
        return False

    return True

def lambda_handler(event, context):
    run_command('/opt/aws --version')
```

Notes on the file system

Code from Lambda layers is unzipped to the /opt directory, so when you invoke a CLI command from a Lambda function, you need to specify the full path /opt/aws.
If you want to write files to Lambda's file system, you can use the /tmp directory, e.g.
run_command('/opt/aws s3 sync s3://BUCKETNAME /tmp')
References
This post would not have been possible without other resources I used to set this up.

Running aws-cli Commands Inside An AWS Lambda Function
Call aws-cli from AWS Lambda
Disclosure
At the time of this writing, I work as a Principal Product Manager at AWS. This post is about my personal project and is not endorsed by AWS.

Tags: aws lambda

Ilya Bezdelev
Jack of all trades, master of some. I write about stuff, code things up, crunch data, compose music, and study psychology. Principal Product Manager @ Amazon Web Services (AWS). Wharton MBA. Ex-DHL. All opinions are my own.

© Ilya Bezdelev, 2016-2020
