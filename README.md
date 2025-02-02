# Update-Frontend-and-Backend-Code-Day-5-
Here's the updated section organized for a GitHub repository:

---

## Download Updated Frontend and Backend Code

### Backup the Existing Folders

```sh
cp -R challenge-day2/ challenge-day2_bkp
cp -R terraform-project/ terraform-project_bkp
```

### Clean Up the Existing Application Files Except the Docker and YAML Files (Backend)

```sh
cd challenge-day2/backend
rm -rf $(find . -mindepth 1 -maxdepth 1 -not \( -name ".*" -o -name Dockerfile -o -name "*.yaml" \))
```

### Clean Up the Existing Application Files Except the Docker and YAML Files (Frontend)

```sh
cd challenge-day2/frontend
rm -rf $(find . -mindepth 1 -maxdepth 1 -not \( -name ".*" -o -name Dockerfile -o -name "*.yaml" \))
```

### Download the Updated Source Code and Unzip It (Backend)

```sh
cd challenge-day2/backend
wget https://tcb-public-events.s3.amazonaws.com/mdac/resources/final/cloudmart-backend-final.zip
unzip cloudmart-backend-final.zip
```

### Download the Updated Source Code and Unzip It (Frontend)

```sh
cd challenge-day2/frontend
wget https://tcb-public-events.s3.amazonaws.com/mdac/resources/final/cloudmart-frontend-final.zip
unzip cloudmart-frontend-final.zip
git add -A
git commit -m "final code"
git push
```

## Google Cloud BigQuery Setup

Follow these steps to set up Google Cloud BigQuery for CloudMart:

### Create a Google Cloud Project

1. Go to the Google Cloud Console here.
2. Click on the project dropdown and select "New Project".
3. Name the project "CloudMart" and create it.

### Enable BigQuery API

1. In the Google Cloud Console, go to "APIs & Services" > "Dashboard".
2. Click "+ ENABLE APIS AND SERVICES".
3. Search for "BigQuery API" and enable it.

### Create a BigQuery Dataset

1. In the Google Cloud Console, go to "BigQuery".
2. In the Explorer pane, click on your project name.
3. Click "CREATE DATASET".
4. Set the Dataset ID to "cloudmart".
5. Choose your data location and click "CREATE DATASET".

### Create a BigQuery Table

1. In the dataset you just created, click "CREATE TABLE".
2. Set the Table name to "cloudmart-orders".
3. Define the schema according to your order structure. For example:
    - id: STRING
    - items: JSON
    - userEmail: STRING
    - total: FLOAT
    - status: STRING
    - createdAt: TIMESTAMP
4. Click "CREATE TABLE".

### Create Service Account and Key

1. In the Google Cloud Console, go to "IAM & Admin" > "Service Accounts".
2. Click "CREATE SERVICE ACCOUNT".
3. Name it "cloudmart-bigquery-sa" and grant it the "BigQuery Data Editor" role.
4. After creating, click on the service account, go to the "Keys" tab, and click "ADD KEY" > "Create new key".
5. Choose JSON as the key type and create.
6. Save the downloaded JSON file as `google_credentials.json`.

### Configure Lambda Function

1. Navigate to the root directory of your project.
2. Enter the Lambda function directory:

    ```sh
    cd challenge-day2/backend/src/lambda/addToBigQuery
    ```

3. Install the required dependencies:

    ```sh
    sudo yum install npm
    npm install
    ```

4. Edit the `google_credentials.json` file in this directory and place the content of your key.
5. Create a zip file of the entire directory:

    ```sh
    zip -r dynamodb_to_bigquery.zip .
    ```

6. This zip file will be used when creating or updating the Lambda function.
7. Return to the root directory of your project:

    ```sh
    cd ../../..
    ```

### Update Lambda Function Environment Variables

Remember to never commit the `google_credentials.json` file to version control. It should be added to your `.gitignore` file.

## Terraform Steps

1. Remove the `main.tf` file and create an empty one:

    ```sh
    rm main.tf
    nano main.tf
    ```

2. Add these Terraform lines to the end of the `main.tf` file to create the Lambda for the BigQuery insert. Update the values in red first.

    ```hcl
    provider "aws" {
      region = "us-east-1"
    }

    # Tables DynamoDB
    resource "aws_dynamodb_table" "cloudmart_products" {
      name           = "cloudmart-products"
      billing_mode   = "PAY_PER_REQUEST"
      hash_key       = "id"

      attribute {
        name = "id"
        type = "S"
      }
    }

    resource "aws_dynamodb_table" "cloudmart_orders" {
      name           = "cloudmart-orders"
      billing_mode   = "PAY_PER_REQUEST"
      hash_key       = "id"

      attribute {
        name = "id"
        type = "S"
      }
      
      stream_enabled   = true
      stream_view_type = "NEW_AND_OLD_IMAGES"
    }

    resource "aws_dynamodb_table" "cloudmart_tickets" {
      name           = "cloudmart-tickets"
      billing_mode   = "PAY_PER_REQUEST"
      hash_key       = "id"

      attribute {
        name = "id"
        type = "S"
      }
    }

    # IAM Role for Lambda function
    resource "aws_iam_role" "lambda_role" {
      name = "cloudmart_lambda_role"

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

    # IAM Policy for Lambda function
    resource "aws_iam_role_policy" "lambda_policy" {
      name = "cloudmart_lambda_policy"
      role = aws_iam_role.lambda_role.id

      policy = jsonencode({
        Version = "2012-10-17"
        Statement = [
          {
            Effect = "Allow"
            Action = [
              "dynamodb:Scan",
              "dynamodb:GetRecords",
              "dynamodb:GetShardIterator",
              "dynamodb:DescribeStream",
              "dynamodb:ListStreams",
              "logs:CreateLogGroup",
              "logs:CreateLogStream",
              "logs:PutLogEvents"
            ]
            Resource = [
              aws_dynamodb_table.cloudmart_products.arn,
              aws_dynamodb_table.cloudmart_orders.arn,
              "${aws_dynamodb_table.cloudmart_orders.arn}/stream/*",
              aws_dynamodb_table.cloudmart_tickets.arn,
              "arn:aws:logs:*:*:*"
            ]
          }
        ]
      })
    }

    # Lambda function for listing products
    resource "aws_lambda_function" "list_products" {
      filename         = "list_products.zip"
      function_name    = "cloudmart-list-products"
      role             = aws_iam_role.lambda_role.arn
      handler          = "index.handler"
      runtime          = "nodejs20.x"
      source_code_hash = filebase64sha256("list_products.zip")

      environment {
        variables = {
          PRODUCTS_TABLE = aws_dynamodb_table.cloudmart_products.name
        }
      }
    }

    # Lambda permission for Bedrock
    resource "aws_lambda_permission" "allow_bedrock" {
      statement_id  = "AllowBedrockInvoke"
      action        = "lambda:InvokeFunction"
      function_name = aws_lambda_function.list_products.function_name
      principal     = "bedrock.amazonaws.com"
    }

    # Output the ARN of the Lambda function
    output "list_products_function_arn" {
      value = aws_lambda_function.list_products.arn
    }

    # Lambda function for DynamoDB to BigQuery
    resource "aws_lambda_function" "dynamodb_to_bigquery" {
      filename         = "../challenge-day2/backend/src/lambda/addToBigQuery/dynamodb_to_bigquery.zip"
      function_name    = "cloudmart-dynamodb-to-bigquery"
      role             = aws_iam_role.lambda_role.arn
      handler          = "index.handler"
      runtime          = "nodejs20.x"
      source_code_hash = filebase64sha256("../challenge-day2/backend/src/lambda/addToBigQuery/dynamodb_to_bigquery.zip")

      environment {
        variables = {
          GOOGLE_CLOUD_PROJECT_ID        = "lustrous-bounty-436219-f1"
          BIGQUERY_DATASET_ID            = "cloudmart"
          BIGQUERY_TABLE_ID              = "cloudmart-orders"
          GOOGLE_APPLICATION_CREDENTIALS = "/var/task/google_credentials.json"
        }
      }
    }

    # Lambda event source mapping for DynamoDB stream
    resource "aws_lambda_event_source_mapping" "dynamodb_stream" {
      event_source_arn  = aws_dynamodb_table.cloudmart_orders.stream_arn
      function_name     = aws_lambda_function.dynamodb_to_bigquery.arn
      starting_position = "LATEST"
    }
    ```

3. Execute `terraform apply` now.

Here's the updated section organized for a GitHub repository:

## Azure Text Analytics Setup

Follow these steps to set up Azure Text Analytics for sentiment analysis:

### Create an Azure Account

1. Go to the Azure portal here.
2. Sign in or create a new account if you don't have one.

### Create a Resource

1. In the Azure portal, click "Create a resource".
2. Search for "Text Analytics" and select it.
3. Click "Create".

### Configure the Resource

1. Choose your subscription and resource group (create a new one if needed).
2. Name the resource (e.g., `cloudmart-text-analytics`).
3. Choose your region and pricing tier.
4. Click "Review + create", then "Create".

### Get the Endpoint and Key

1. Once the resource is created, go to its overview page.
2. In the left menu, under "Resource Management", click "Keys and Endpoint".
3. Copy the endpoint URL and one of the keys.

### Deploy the Changes on Backend

1. Open the file `cloudmart-backend.yaml`:

    ```sh
    nano cloudmart-backend.yaml
    ```

2. Content of `cloudmart-backend.yaml`:

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: cloudmart-backend-app
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: cloudmart-backend-app
      template:
        metadata:
          labels:
            app: cloudmart-backend-app
        spec:
          serviceAccountName: cloudmart-pod-execution-role
          containers:
          - name: cloudmart-backend-app
            image: public.ecr.aws/l4c0j8h9/cloudmart-backend:latest
            env:
            - name: PORT
              value: "5000"
            - name: AWS_REGION
              value: "us-east-1"
            - name: BEDROCK_AGENT_ID
              value: "xxxx"
            - name: BEDROCK_AGENT_ALIAS_ID
              value: "xxxx"
            - name: OPENAI_API_KEY
              value: "xxxx"
            - name: OPENAI_ASSISTANT_ID
              value: "xxxx"
            - name: AZURE_ENDPOINT
              value: "xxxx"
            - name: AZURE_API_KEY
              value: "xxxx"
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: cloudmart-backend-app-service
    spec:
      type: LoadBalancer
      selector:
        app: cloudmart-backend-app
      ports:
        - protocol: TCP
          port: 5000
          targetPort: 5000
    ```

### Build a New Image

Follow the ECR steps to build a new image.

### Update the Deployment on Kubernetes

```sh
kubectl apply -f cloudmart-backend.yaml
```

---
