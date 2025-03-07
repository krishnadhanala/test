### generic_lambda.py
import boto3
import subprocess
import os
import json

def download_from_s3(s3_path, local_path):
    s3 = boto3.client('s3')
    bucket, key = s3_path.replace("s3://", "").split("/", 1)
    s3.download_file(bucket, key, local_path)

def install_requirements(requirements_path):
    subprocess.run(['pip', 'install', '-r', requirements_path], check=True)

def execute_script(script_path):
    subprocess.run(['python3', script_path], check=True)

def lambda_handler(event, context):
    try:
        s3_script_path = event["script_path"]
        s3_requirements_path = event.get("requirements_path")
        
        local_script_path = "/tmp/input_script.py"
        local_requirements_path = "/tmp/requirements.txt"
        
        download_from_s3(s3_script_path, local_script_path)
        
        if s3_requirements_path:
            download_from_s3(s3_requirements_path, local_requirements_path)
            install_requirements(local_requirements_path)
        
        execute_script(local_script_path)
        
        return {"statusCode": 200, "body": "Script executed successfully!"}
    except Exception as e:
        return {"statusCode": 500, "body": f"Error: {str(e)}"}

### scheduler_lambda.py
import boto3
import os
import json
import uuid

def schedule_lambda(script_s3_path, requirements_s3_path, schedule_rate):
    eventbridge = boto3.client('events')
    lambda_client = boto3.client('lambda')
    rule_name = f"scheduled-exec-{uuid.uuid4()}"
    
    response = eventbridge.put_rule(
        Name=rule_name,
        ScheduleExpression=f"rate({schedule_rate} minutes)",
        State="ENABLED"
    )
    
    eventbridge.put_targets(
        Rule=rule_name,
        Targets=[{
            'Id': '1',
            'Arn': os.environ.get("GENERIC_LAMBDA_ARN"),
            'Input': json.dumps({"script_path": script_s3_path, "requirements_path": requirements_s3_path})
        }]
    )
    
    return rule_name

def lambda_handler(event, context):
    try:
        rule_name = schedule_lambda(event["script_path"], event.get("requirements_path"), event["schedule_rate"])
        return {"statusCode": 200, "body": f"Lambda scheduled successfully with rule: {rule_name}"}
    except Exception as e:
        return {"statusCode": 500, "body": f"Error: {str(e)}"}



#test.py
import boto3
import os

def lambda_test_script():
    s3 = boto3.client('s3')
    bucket_name = os.environ.get("TEST_BUCKET_NAME")  # Set this environment variable in Lambda
    file_key = "lambda_execution_test.txt"

    s3.put_object(
        Bucket=bucket_name,
        Key=file_key,
        Body="Executed"
    )

if __name__ == "__main__":
    lambda_test_script()
