## Step 1: Set Up Your S3 Bucket

### 1.1 Create the S3 Bucket

1. Navigate to the S3 service in your AWS console
2. Click "Create bucket"
3. Enter a unique bucket name (e.g., `your-name-file-storage-lab-[random-number]`)
4. Choose your preferred region
5. Under "Block Public Access settings," keep all options checked
7. Click "Create bucket"

### 1.2 Configure CORS (Cross-Origin Resource Sharing)

1. Select your newly created bucket
2. Go to the "Permissions" tab
3. Scroll down to "Cross-origin resource sharing (CORS)"
4. Click "Edit" and add the following configuration:

```json
[
    {
        "AllowedHeaders": ["*"],
        "AllowedMethods": ["GET", "PUT", "POST", "DELETE"],
        "AllowedOrigins": ["*"],
        "ExposeHeaders": []
    }
]
```

## Step 2: Create Lambda Functions

### 2.1 Create File Upload Lambda Function

1. Navigate to Lambda service
2. Click "Create function"
3. Choose "Author from scratch"
4. Function name: `file-upload-function`
5. Runtime: Python 3.9 or later
6. Under "Change default execution role," select "Use an existing role"
7. Choose **LabRole** from the dropdown
8. Click "Create function"

### 2.2 Upload Function Code

Replace the default code with:

```python
import json
import boto3
import base64
from botocore.exceptions import ClientError

s3_client = boto3.client('s3')
BUCKET_NAME = 'your-bucket-name-here'  # Replace with your bucket name

def lambda_handler(event, context):
    try:
        # Parse the request body
        body = json.loads(event['body']) if event.get('body') else {}
        
        # Get filename and file content
        filename = body.get('filename')
        file_content = body.get('content')
        
        if not filename or not file_content:
            return {
                'statusCode': 400,
                'headers': {
                    'Content-Type': 'application/json',
                    'Access-Control-Allow-Origin': '*'
                },
                'body': json.dumps({
                    'error': 'Missing filename or content'
                })
            }
        
        # Decode base64 content
        try:
            decoded_content = base64.b64decode(file_content)
        except:
            return {
                'statusCode': 400,
                'headers': {
                    'Content-Type': 'application/json',
                    'Access-Control-Allow-Origin': '*'
                },
                'body': json.dumps({
                    'error': 'Invalid base64 content'
                })
            }
        
        # Upload to S3
        s3_client.put_object(
            Bucket=BUCKET_NAME,
            Key=filename,
            Body=decoded_content
        )
        
        return {
            'statusCode': 200,
            'headers': {
                'Content-Type': 'application/json',
                'Access-Control-Allow-Origin': '*'
            },
            'body': json.dumps({
                'message': f'File {filename} uploaded successfully'
            })
        }
        
    except ClientError as e:
        return {
            'statusCode': 500,
            'headers': {
                'Content-Type': 'application/json',
                'Access-Control-Allow-Origin': '*'
            },
            'body': json.dumps({
                'error': f'S3 error: {str(e)}'
            })
        }
    except Exception as e:
        return {
            'statusCode': 500,
            'headers': {
                'Content-Type': 'application/json',
                'Access-Control-Allow-Origin': '*'
            },
            'body': json.dumps({
                'error': f'Unexpected error: {str(e)}'
            })
        }
```

**Important:** Replace `your-bucket-name-here` with your actual bucket name.

### 2.3 Create File Download Lambda Function

1. Create another function named `file-download-function`
2. Use the same configuration as the upload function (select **LabRole**)
3. Replace the code with:

```python
import json
import boto3
import base64
from botocore.exceptions import ClientError

s3_client = boto3.client('s3')
BUCKET_NAME = 'your-bucket-name-here'  # Replace with your bucket name

def lambda_handler(event, context):
    try:
        # Get filename from path parameters
        filename = event['pathParameters']['filename'] if event.get('pathParameters') else None
        
        if not filename:
            return {
                'statusCode': 400,
                'headers': {
                    'Content-Type': 'application/json',
                    'Access-Control-Allow-Origin': '*'
                },
                'body': json.dumps({
                    'error': 'Missing filename parameter'
                })
            }
        
        # Download from S3
        response = s3_client.get_object(Bucket=BUCKET_NAME, Key=filename)
        file_content = response['Body'].read()
        
        # Encode content as base64
        encoded_content = base64.b64encode(file_content).decode('utf-8')
        
        return {
            'statusCode': 200,
            'headers': {
                'Content-Type': 'application/json',
                'Access-Control-Allow-Origin': '*'
            },
            'body': json.dumps({
                'filename': filename,
                'content': encoded_content,
                'content_type': response.get('ContentType', 'application/octet-stream')
            })
        }
        
    except ClientError as e:
        if e.response['Error']['Code'] == 'NoSuchKey':
            return {
                'statusCode': 404,
                'headers': {
                    'Content-Type': 'application/json',
                    'Access-Control-Allow-Origin': '*'
                },
                'body': json.dumps({
                    'error': f'File {filename} not found'
                })
            }
        else:
            return {
                'statusCode': 500,
                'headers': {
                    'Content-Type': 'application/json',
                    'Access-Control-Allow-Origin': '*'
                },
                'body': json.dumps({
                    'error': f'S3 error: {str(e)}'
                })
            }
    except Exception as e:
        return {
            'statusCode': 500,
            'headers': {
                'Content-Type': 'application/json',
                'Access-Control-Allow-Origin': '*'
            },
            'body': json.dumps({
                'error': f'Unexpected error: {str(e)}'
            })
        }
```

### 2.4 Create File List Lambda Function

Create a third function named `file-list-function` using **LabRole** with this code:

```python
import json
import boto3
from botocore.exceptions import ClientError

s3_client = boto3.client('s3')
BUCKET_NAME = 'your-bucket-name-here'  # Replace with your bucket name

def lambda_handler(event, context):
    try:
        # List objects in S3 bucket
        response = s3_client.list_objects_v2(Bucket=BUCKET_NAME)
        
        files = []
        if 'Contents' in response:
            for obj in response['Contents']:
                files.append({
                    'filename': obj['Key'],
                    'size': obj['Size'],
                    'last_modified': obj['LastModified'].isoformat()
                })
        
        return {
            'statusCode': 200,
            'headers': {
                'Content-Type': 'application/json',
                'Access-Control-Allow-Origin': '*'
            },
            'body': json.dumps({
                'files': files
            })
        }
        
    except ClientError as e:
        return {
            'statusCode': 500,
            'headers': {
                'Content-Type': 'application/json',
                'Access-Control-Allow-Origin': '*'
            },
            'body': json.dumps({
                'error': f'S3 error: {str(e)}'
            })
        }
    except Exception as e:
        return {
            'statusCode': 500,
            'headers': {
                'Content-Type': 'application/json',
                'Access-Control-Allow-Origin': '*'
            },
            'body': json.dumps({
                'error': f'Unexpected error: {str(e)}'
            })
        }
```

## Step 3: Create API Gateway

### 3.1 Create REST API

1. Navigate to API Gateway service
2. Click "Create API"
3. Choose "REST API" (not private)
4. Click "Build"
5. Choose "New API"
6. API name: `file-management-api`
7. Description: `API for file upload and download operations`
8. Click "Create API"

### 3.2 Create Resources and Methods

#### Create Upload Endpoint

1. Click "Actions" → "Create Resource"
2. Resource Name: `upload`
3. Resource Path: `/upload`
4. Click "Create Resource"

5. Select the `/upload` resource
6. Click "Actions" → "Create Method"
7. Choose "POST" from dropdown and click the checkmark
8. Integration type: Lambda Function
9. Use Lambda Proxy integration: ✓ (checked)
10. Lambda Region: Your region
11. Lambda Function: `file-upload-function`
12. Click "Save" and "OK" to give API Gateway permission

#### Create Download Endpoint

1. Click "Actions" → "Create Resource"
2. Resource Name: `download`
3. Resource Path: `/download`
4. Click "Create Resource"

5. Select the `/download` resource
6. Click "Actions" → "Create Resource" (sub-resource)
7. Resource Name: `filename`
8. Resource Path: `{filename}`
9. Click "Create Resource"

10. Select the `/{filename}` resource
11. Click "Actions" → "Create Method"
12. Choose "GET" and click checkmark
13. Integration type: Lambda Function
14. Use Lambda Proxy integration: ✓ (checked)
15. Lambda Function: `file-download-function`
16. Click "Save" and "OK"

#### Create List Files Endpoint

1. Click "Actions" → "Create Resource"
2. Resource Name: `files`
3. Resource Path: `/files`
4. Click "Create Resource"

5. Select the `/files` resource
6. Click "Actions" → "Create Method"
7. Choose "GET" and click checkmark
8. Integration type: Lambda Function
9. Use Lambda Proxy integration: ✓ (checked)
10. Lambda Function: `file-list-function`
11. Click "Save" and "OK"

### 3.3 Enable CORS

For each method (POST /upload, GET /download/{filename}, GET /files):

1. Select the method
2. Click "Actions" → "Enable CORS"
3. Keep default settings
4. Click "Enable CORS and replace existing CORS headers"

### 3.4 Deploy the API

1. Click "Actions" → "Deploy API"
2. Deployment stage: New Stage
3. Stage name: `prod`
4. Click "Deploy"
5. Note the Invoke URL - you'll need this for testing

## Step 4: Test Your API

### 4.1 Test File Upload

Use a tool like Postman or curl to test. First, encode your test content to base64:

```bash
echo "Hello World!" | base64
```

This should return: `SGVsbG8gV29ybGQhCg==`

Then test the upload:

```bash
curl -X POST https://your-api-id.execute-api.region.amazonaws.com/prod/upload \
  -H "Content-Type: application/json" \
  -d '{
    "filename": "test.txt",
    "content": "SGVsbG8gV29ybGQhCg=="
  }'
```

### 4.2 Test File List

```bash
curl https://your-api-id.execute-api.region.amazonaws.com/prod/files
```

### 4.3 Test File Download

```bash
curl https://your-api-id.execute-api.region.amazonaws.com/prod/download/test.txt
```

## Step 5: Monitor and Debug

### 5.1 Check CloudWatch Logs

1. Navigate to CloudWatch
2. Click "Log groups"
3. Find logs for your Lambda functions (`/aws/lambda/function-name`)
4. Review execution logs for any errors

### 5.2 Test in API Gateway Console

1. Go to API Gateway console
2. Select your API
3. Choose a method
4. Click "TEST"
5. For POST /upload, use this test body:
```json
{
  "filename": "test.txt",
  "content": "SGVsbG8gV29ybGQhCg=="
}
```

## Troubleshooting Common Issues

**Lambda timeout errors:** Increase timeout in Lambda function configuration (Configuration → General configuration → Edit → Timeout)

**Permission errors:** The LabRole should have sufficient permissions, but if you encounter issues, check CloudWatch logs for specific error messages

**CORS errors:** Ensure CORS is enabled on all methods and the OPTIONS method is created automatically

**Base64 encoding issues:** Make sure file content is properly base64 encoded for upload. You can use online base64 encoders or command line tools

**API Gateway Integration:** Make sure "Use Lambda Proxy integration" is checked when setting up the methods



## Extension Ideas

- Add file deletion functionality (DELETE method)
- Implement file type validation in Lambda
- Add file size limits
- Add error handling for large files
