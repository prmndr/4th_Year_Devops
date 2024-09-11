# Practical Lab 4: Creating an AWS Lambda Function with Terraform

## Objective
In this lab, you will learn how to use Terraform to create and deploy a simple Python-based AWS Lambda function. You'll set up the necessary AWS resources and use Terraform to manage the infrastructure.

## Prerequisites
- AWS account with appropriate permissions
- AWS CLI installed and configured with your credentials
- Terraform installed on your local machine
- Basic understanding of Python, AWS, and command-line operations

## Estimated Time
1 hour

## Use Case
You're a DevOps engineer tasked with creating a serverless function that responds to HTTP requests. The function should return a simple greeting message. You need to set this up in a way that's easily reproducible and manageable, hence the use of Terraform for infrastructure provisioning.

## Steps

### 1. Set up the project structure

1. Create a new directory for your project:
   ```
   mkdir lambda-terraform-lab
   cd lambda-terraform-lab
   ```

2. Create the following file structure:
   ```
   lambda-terraform-lab/
   ├── main.tf
   ├── variables.tf
   ├── outputs.tf
   └── src/
       └── lambda_function.py
   ```

### 2. Create the Lambda function code

Create `src/lambda_function.py` with the following content:

```python
import json

def lambda_handler(event, context):
    return {
        'statusCode': 200,
        'body': json.dumps('Hello from AWS Lambda!')
    }
```

### 3. Create the Terraform configuration files

1. Create `variables.tf`:

```hcl
variable "aws_region" {
  description = "The AWS region to create resources in"
  default     = "us-west-2"
}

variable "lambda_function_name" {
  description = "Name of the Lambda function"
  default     = "HelloWorldFunction"
}
```

2. Create `main.tf`:

```hcl
provider "aws" {
  region = var.aws_region
}

data "archive_file" "lambda_zip" {
  type        = "zip"
  source_dir  = "${path.module}/src"
  output_path = "${path.module}/lambda_function.zip"
}

resource "aws_lambda_function" "hello_world" {
  filename      = "lambda_function.zip"
  function_name = var.lambda_function_name
  role          = aws_iam_role.lambda_exec.arn
  handler       = "lambda_function.lambda_handler"
  runtime       = "python3.8"

  source_code_hash = data.archive_file.lambda_zip.output_base64sha256
}

resource "aws_iam_role" "lambda_exec" {
  name = "lambda_exec_role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "lambda.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_policy" {
  role       = aws_iam_role.lambda_exec.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

resource "aws_apigatewayv2_api" "lambda_api" {
  name          = "lambda-http-api"
  protocol_type = "HTTP"
}

resource "aws_apigatewayv2_stage" "lambda_stage" {
  api_id = aws_apigatewayv2_api.lambda_api.id
  name   = "$default"
  auto_deploy = true
}

resource "aws_apigatewayv2_integration" "lambda_integration" {
  api_id           = aws_apigatewayv2_api.lambda_api.id
  integration_type = "AWS_PROXY"

  integration_uri    = aws_lambda_function.hello_world.invoke_arn
  integration_method = "POST"
}

resource "aws_apigatewayv2_route" "lambda_route" {
  api_id    = aws_apigatewayv2_api.lambda_api.id
  route_key = "GET /hello"

  target = "integrations/${aws_apigatewayv2_integration.lambda_integration.id}"
}

resource "aws_lambda_permission" "api_gw" {
  statement_id  = "AllowExecutionFromAPIGateway"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.hello_world.function_name
  principal     = "apigateway.amazonaws.com"

  source_arn = "${aws_apigatewayv2_api.lambda_api.execution_arn}/*/*"
}
```

3. Create `outputs.tf`:

```hcl
output "function_name" {
  description = "Name of the Lambda function"
  value       = aws_lambda_function.hello_world.function_name
}

output "base_url" {
  description = "Base URL for API Gateway stage"
  value       = aws_apigatewayv2_stage.lambda_stage.invoke_url
}
```

### 4. Initialize and apply the Terraform configuration

1. Initialize Terraform:
   ```
   terraform init
   ```

2. Review the planned changes:
   ```
   terraform plan
   ```

3. Apply the configuration:
   ```
   terraform apply
   ```

   When prompted, type `yes` to confirm the changes.

### 5. Test the Lambda function

1. Once Terraform has finished applying the changes, note the `base_url` output.

2. Use `curl` or a web browser to test the function:
   ```
   curl <base_url>/hello
   ```

   You should see the response: `"Hello from AWS Lambda!"`

### 6. Modify the Lambda function

1. Update the `src/lambda_function.py` file:

```python
import json

def lambda_handler(event, context):
    return {
        'statusCode': 200,
        'body': json.dumps('Hello from AWS Lambda! This function has been updated.')
    }
```

2. Re-run Terraform apply:
   ```
   terraform apply
   ```

3. Test the updated function again using the same URL.

### 7. Clean up

When you're done experimenting, destroy the created resources:

```
terraform destroy
```

When prompted, type `yes` to confirm.

## Conclusion
In this lab, you've learned how to use Terraform to create and manage an AWS Lambda function with an API Gateway. This demonstrates the power of Infrastructure as Code for managing cloud resources and serverless applications.

## Additional Exercises
1. Add environment variables to your Lambda function using Terraform.
2. Create a CloudWatch Log Group for your Lambda function and configure Terraform to manage it.
3. Modify the Lambda function to accept and use query parameters from the API Gateway.
4. Set up a CloudWatch Event to trigger your Lambda function on a schedule.

Remember to explore the AWS Lambda and Terraform documentation for more advanced features and best practices in serverless computing and infrastructure management.
