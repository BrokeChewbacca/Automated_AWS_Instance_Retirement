# Automated_AWS_Instance_Retirement

## Description
- These three scripts automate the stopping and starting of instances that have been marked for instance retirement within a specified maintenance window.
- InstanceRetirement-Main.py is triggered by the 'InstanceRetirement-Main' CloudWatch event rule at a specified interval.
  - When triggered it check all instances in the AWS account to see if they have a health event of ‘instance-retirement’.
  -	If no instances have this event, nothing happens.
  -	If at least one instance is found with this health event, it checks to see if the current time is within the specified maintenance window.
    - If true: the instance is stopped, an SQS message is sent to the 'InstanceRetirement-InstanceStartQueue' SQS queue, and the 'InstanceRetirement-StartInstance' CloudWatch event rule is enabled. 
    - If false: an SQS message is sent to the 'InstanceRetirement-InstanceStopQueue' SQS queue, and the CloudWatch event rule 'InstanceRetirement-StopInstance' is enabled. 
- InstanceRetirement-StopInstance.py is triggered by the 'InstanceRetirement-StopInstance' CloudWatch event rule. This rule will only be active when enabled by the 'InstanceRetirement-Main' script.
  - When triggered, it polls the 'InstanceRetirement-InstanceStopQueue' SQS queue for messages.
    - If no messages are found, the script ends and runs again at the next interval.
    - If a message is found, it checks to see if the current time is within the specified maintenance window.
      - If true: the instance is stopped, the SQS message is deleted, an SQS message is sent to the 'InstanceRetirement-InstanceStartQueue' SQS queue , the 'InstanceRetirement-StopInstance' CloudWatch event rule gets disabled, and the 'InstanceRetirement-StartInstance' CloudWatch event rule gets enabled.
      - If false: the SQS message is returned to the queue, the script ends, and is run again at the next interval specified by the 'InstanceRetirement-StartInstance' CloudWatch event rule.
- InstanceRetirement-StartInstance.py is triggered by the 'InstanceRetirement-StartInstance' CloudWatch event rule. This rule will only be active when enabled by either the 'InstanceRetirement-Main' or 'InstanceRetirement-StopInstance' scripts.
  - When triggered, it polls the 'InstanceRetirement-InstanceStartQueue' SQS queue for messages.
    - If no messages are found, the script ends and runs again at the next interval.
    - If a message is found, it checks to see if the instance is in the ‘stopped’ state.
      - If true: the instance is started, the SQS message is deleted, and the 'InstanceRetirement-StartInstance' CloudWatch event rule is disabled.
      - If false: the SQS message is returned to the queue, the script ends, and is run again at the next interval specified by the 'InstanceRetirement-StartInstance' CloudWatch event rule.


## AWS Environment Configuration
- IAM
  - Policies
    - Create a policy named ‘InstanceRetirement’ using the ‘InstanceRetirement IAM Policy.json’ file in the ‘IAM Policies’ folder.
    - Create a policy name ‘LambdaUpdateFunctionCode’ using the ‘LambdaUpdateFunctionCode IAM Policy.json’ file in the ‘IAM Policies’ folder.
  - Roles
    - Create a role named ‘InstanceRetirement’ that will be used by Lambda. For permissions give it the ‘InstanceRetirement’ policy.
  - Users
    - Create a user named ‘LambdaUpdateFunctionCodeCLI’ with programmatic access. Attach the existing ‘LambdaUpdateFunctionCode’ policy.
      - MAKE SURE YOU SAVE THE CREDS FOR THIS USER
- SQS
  - Create two standard queues named ‘InstanceRetirement-InstanceStartQueue’ and ‘InstanceRetirement-InstanceStopQueue’.
    - All you need to configure is the queue name and verify it is a standard queue. After that you can click the ‘Quick-Create Queue’ button.
- Lambda
  - Create a function named ‘InstanceRetirement-Main’
    - Runtime: Python 3.7
    - Permissions: Existing Role - InstanceRetirement
    - Don’t worry about the triggers now, that will be set in the next section automatically.
    - Under ‘Function code’, set the Handler to ‘InstanceRetirement-Main.lambda_handler’
      - Don’t worry about the function code right now, you’ll upload it later using the CLI.
    - Under ‘Environment variables’ setup the following:
      - MaintWindowStart_ST
        - This is the start of the maintenance window (when the instances can stop) in standard time. The time needs to be in UTC and in the format HH:MM:SS
      - MaintWindowEnd_ST
        - This is the end of the maintenance window (when the instances can stop) in standard time. The time needs to be in UTC and in the format HH:MM:SS
      - MaintWindowStart_DST
        - This is the start of the maintenance window (when the instances can stop) in daylight-savings time. The time needs to be in UTC and in the format HH:MM:SS
      - MaintWindowEnd_DST
        - This is the end of the maintenance window (when the instances can stop) in daylight-savings time. The time needs to be in UTC and in the format HH:MM:SS
      - TimeZone
        - This is the time zone that you want these maintenance window times to reference. This is needed to determine if it is standard time or daylight-saving time.
        - You can find the correct time zone formats for python [here](https://gist.github.com/heyalexej/8bf688fd67d7199be4a1682b3eec7568)
  - Create a function named ‘InstanceRetirement-StopInstance’
    - Runtime: Python 3.7
    - Permissions: Existing Role - InstanceRetirement
    - Don’t worry about the triggers now, that will be set in the next section automatically.
    - Under ‘Function code’, set the Handler to ‘InstanceRetirement-StopInstance.lambda_handler’
      - Don’t worry about the function code right now, you’ll upload it later using the CLI.
  - Create a function named ‘InstanceRetirement-StartInstance’
    - Runtime: Python 3.7
    - Permissions: Existing Role - InstanceRetirement
    - Don’t worry about the triggers now, that will be set in the next section automatically.
    - Under ‘Function code’, set the Handler to ‘InstanceRetirement-StartInstance.lambda_handler’
      - Don’t worry about the function code right now, you’ll upload it later using the CLI.
- CloudWatch Events Rules
  - Create the following CloudWatch events rules:
    - InstanceRetirement-Main
      - Schedule: Fixed rate of X hours (set X equal to how often you want to check the instances health events)
      - Targets: Lambda function: InstanceRetirement-Main
      - Enabled: True
    - InstanceRetirement-StopInstance
      - Schedule: Fixed rate of 5 minutes
      - Targets: Lambda function: InstanceRetirement-StopInstance
      - Enabled: False
    - InstanceRetirement-StartInstance
      - Schedule: Fixed rate of 5 minutes
      - Targets: Lambda function: InstanceRetirement-StartInstance
      - Enabled: False
      
      
## Required Code Changes
- All three scripts require some minor changes because 1) the SQS URLs change and 2) you may name the CloudWatch Event Rules differently.
- InstanceRetirement-Main.py
  - Lines 28-31
  - For the ‘stopQueueUrl’ and ‘startQueueUrl’ variables, set them equal to the ‘InstanceRetirement-InstanceStopQueue’ and ‘InstanceRetirement-InstanceStartQueue’ SQS queue URL’s respectively. These URLs need to be strings (i.e. they need to be surrounded by a pair of apostrophes).
  - For the ‘stopRule’ and ‘startRule’ variables, set them equal to the ‘InstanceRetirement-InstanceStop’ and ‘InstanceRetirement-InstanceStart’ CloudWatch event rule names respectively. These values need to be strings (i.e. they need to be surrounded by a pair of apostrophes). 
- InstanceRetirement-StopInstance.py
  - Lines 19-22
  - For the ‘stopQueueUrl’ and ‘startQueueUrl’ variables, set them equal to the ‘InstanceRetirement-InstanceStopQueue’ and ‘InstanceRetirement-InstanceStartQueue’ URL’s respectively. These values need to be strings (i.e. they need to be surrounded by a pair of apostrophes).
  - For the ‘stopRule’ and ‘startRule’ variables, set them equal to the ‘InstanceRetirement-InstanceStop’ and ‘InstanceRetirement-InstanceStart’ CloudWatch event rule names respectively. These values need to be strings (i.e. they need to be surrounded by a pair of apostrophes). 
- InstanceRetirement-StartInstance.py
  - Lines 18-19
  - For the ‘startQueueUrl’ variable, set it equal to the ‘InstanceRetirement-InstanceStartQueue’ URL. This value needs to be a string (i.e. it needs to be surrounded by a pair of apostrophes).
  - For the ‘startRule’ variable, set it equal to the ‘InstanceRetirement-InstanceStart’ CloudWatch event rule name. This value needs to be a string (i.e. it needs to be surrounded by a pair of apostrophes). 


## Upload the code:
- **Only perform these next steps after everything in the ‘AWS Environment Configuration’ and ‘Required Code Changes’ have been completed.**
- You will need the AWS CLI installed on the system you perform the following steps with.
- Recommended to do the following steps on a Linux or Unix system otherwise, some of the CLI commands may not be recognized.
- The following assumes you have named everything exactly as told to above. If you have changed any of the script names or Lambda function names, the commands below will need to be tweaked accordingly. I’m also assuming you have each of the three folders containing the scripts on your systems desktop.
- Open terminal and type `cd desktop`
- Type `aws configure` and enter the ‘AWS Access Key ID’ and ‘AWS Secret Access Key’ for the LambdaUpdateFunctionCodeCLI user created earlier. Enter the region name you are operating in. Leave the ‘Default output format’ as ‘None’.
  - Upload InstanceRetirement-Main
    - `cd InstanceRetirement-Main`
    - `cd package`
    - `zip -r9 ../InstanceRetirement-Main.zip .`
    - `cd ../`
    - `zip -g InstanceRetirement-Main.zip InstanceRetirement-Main.py`
    - `aws lambda update-function-code --function-name InstanceRetirement-Main --zip-file fileb://InstanceRetirement-Main.zip`
    - `cd ../`
  - Upload InstanceRetirement-StopInstance
    - `cd InstanceRetirement-StopInstance`
    - `zip -r9 InstanceRetirement-StopInstance.zip .`
    - `aws lambda update-function-code --function-name InstanceRetirement-StopInstance --zip-file fileb://InstanceStopInstance.zip`
    - `cd ../`
  - Upload InstanceRetirement-StartInstance
    - `cd InstanceRetirement-StartInstance`
    - `zip -r9 InstanceRetirement-StartInstance.zip .`
    - `aws lambda update-function-code --function-name InstanceRetirement-StartInstance --zip-file fileb://InstanceStartInstance.zip`