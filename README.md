## Step 1: Set Up Your S3 Bucket

### 1.1 Create the S3 Bucket

1. Navigate to the S3 service in your AWS console
2. Click "Create bucket"
3. Enter a unique bucket name (e.g., `your-name-file-storage-lab-[random-number]`)
4. Under "Block Public Access settings," keep all options checked
5. Click "Create bucket"

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
5. Runtime: Python 3.13 or later
6. Under "Change default execution role," select "Use an existing role"
7. Choose **LabRole** from the dropdown
8. Click "Create function"

### 2.2 Upload Function Code

1. Replace the default code with:

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

2. Then click the "Deploy" button 

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
**Important:** Replace `your-bucket-name-here` with your actual bucket name.

4. Then click the "Deploy" button
### 2.4 Create File List Lambda Function

1. Create a third function named `file-list-function` using **LabRole** with this code:

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
**Important:** Replace `your-bucket-name-here` with your actual bucket name.

2. Then click the "Deploy" button

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

1. Click "Create Resource"
2. Resource Name: `upload`
3. Click "Create Resource"

5. Select the `/upload` resource
6. Click "Create Method"
7. Choose "POST" from dropdown
8. Integration type: Lambda Function
9. Use Lambda Proxy integration
10. Lambda Region: Your region
11. Lambda Function: `file-upload-function`
12. Click "Create Method"

#### Create Download Endpoint

1. Click "Create Resource"
2. Resource Name: `download`
3. Resource Path: `/`
4. Click "Create Resource"

5. Select the `/download` resource
6. Click "Create Resource" 
7. Resource Name: `{filename}`
8. Resource Path: `/download`
9. Click "Create Resource"

10. Select the `/{filename}` resource
11. Click "Create Method"
12. Choose "GET" 
13. Integration type: Lambda Function
14. Use Lambda Proxy integration
15. Lambda Function: `file-download-function`
16. Click "Create Method"

#### Create List Files Endpoint

1. Click "Actions" → "Create Resource"
2. Resource Name: `files`
3. Resource Path: `/`
4. Click "Create Resource"

5. Select the `/files` resource
6. Click "Create Method"
7. Choose "GET"
8. Integration type
9. Use Lambda Proxy integration
10. Lambda Function: `file-list-function`
11. Click "Create Method"

### 3.3 Enable CORS

For each method (POST /upload, GET /download/{filename}, GET /files):

1. Select the method
2. Click "Actions" → "Enable CORS"
3. Keep default settings
4. Click "Save"
### 3.4 Deploy the API

1. Click "Deploy API"
2. Deployment stage: New Stage
3. Stage name: `prod`
4. Click "Deploy"
5. Copy the Invoke URL 

## Step 4: Test Your API Using Postman

### 4.1 Get Your API Base URL

### 4.2 Test File Upload

1. Open Postman and create a new request
2. Set method to **POST**
3. Enter URL: `https://your-api-id.execute-api.region.amazonaws.com/prod/upload`
4. In the **Headers** tab, add:
   - Key: `Content-Type`
   - Value: `application/json`
5. In the **Body** tab, select **raw** and **JSON**, then enter:

```json
{
  "filename": "test.txt",
  "content": "SGVsbG8gV29ybGQh"
}
```

6. Click **Send**

The content "SGVsbG8gV29ybGQh" is "Hello World!" encoded in base64.

### 4.3 Test File List

1. Create a new request in Postman
2. Set method to **GET**
3. Enter URL: `https://your-api-id.execute-api.region.amazonaws.com/prod/files`
4. Click **Send**

### 4.4 Test File Download

1. Create a new request in Postman
2. Set method to **GET**
3. Enter URL: `https://your-api-id.execute-api.region.amazonaws.com/prod/download/test.txt`
4. Click **Send**

**Note:** Replace `your-api-id.execute-api.region.amazonaws.com` with your actual API Gateway URL from Step 3.4.

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
