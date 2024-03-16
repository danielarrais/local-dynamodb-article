# How to create and initialize tables in a local DynamoDB with docker compose

Today, I established a local environment for a Java microservice that interacts with AWS DynamoDB. While it's quite straightforward to create a local DynamoDB instance using Docker, the real challenge lay in initializing my tables using a Docker Compose service. Thankfully, I found a solution.

Initially, I set up the DynamoDB service in my Docker Compose file:

```yaml
version: '3'
services:
  dynamodb-local:
    command: "-jar DynamoDBLocal.jar -sharedDb -dbPath ./data"
    image: "amazon/dynamodb-local:latest"
    user: root
    ports:
      - "8000:8000"
    volumes:
      - "./docker/dynamodb:/home/dynamodblocal/data"
    working_dir: /home/dynamodblocal
```

This code doesn't have any secrets; I extracted it from [here](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DynamoDBLocal.DownloadingAndRunning.html). However, I’ll initialize the tables in another service, so it's useful to create a health check for our `dynamodb-local` service:

```yaml
version: '3'
services:
  dynamodb-local:
    command: "-jar DynamoDBLocal.jar -sharedDb -dbPath ./data"
    image: "amazon/dynamodb-local:latest"
    user: root
    ports:
      - "8000:8000"
    volumes:
      - "./docker/dynamodb:/home/dynamodblocal/data"
    working_dir: /home/dynamodblocal
    healthcheck:
      test: ["CMD-SHELL", '[ "$(curl -s -o /dev/null -I -w ''%{http_code}'' http://localhost:8000)" == "400" ]']
      interval: 10s
      timeout: 10s
      retries: 10
```

This health check test confirms that when curl tries to access the Dynamo port, it returns an HTTP code of 400, which is the expected code. Now, I can proceed to create the next service that will be responsible for creating our tables.

## The solution

Now that i've created the Dynamo service, i'll be implementing the final solution. In the solution, i need to create files with the table descriptions (you can read more about it [here](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_DescribeTable.html)). By using files, it's easier to pass the descriptions in a terminal command. The command is `aws dynamodb create-table --endpoint-url "<http://localhost:8000>" --cli-input-json file://file`. With this command, you can create a table in a local dynamodb.

For this article, I created a simple table with only a single field, called ID, and I placed it inside a folder called schemas. The folder will be used to copy the schemas into the container. Below, you can see the table description:

```json
{
  "TableName": "MyDynamoTable",
  "KeySchema": [
    {
      "AttributeName": "ID",
      "KeyType": "HASH"
    }
  ],
  "AttributeDefinitions": [
    {
      "AttributeName": "ID",
      "AttributeType": "N"
    }
  ],
  "ProvisionedThroughput": {
    "ReadCapacityUnits": 5,
    "WriteCapacityUnits": 5
  }
}
```

I'll need to use the AWS CLI by creating a Docker service with the `amazon/aws-cli` image. In this service, i'll replace the default entrypoint with another one. Although I can run the presented command with the default entrypoint (`aws`), it only allows creating a single table, and my goal is to create many tables. For this, the new entrypoint will be the `bash`, and i'll use a shell loop to run the command for each JSON file. Below, you can see the new compose file:

```yaml
version: '3'
services:
  dynamodb-local:
  ... 
  dynamodb-local-setup:
    depends_on:
      dynamodb-local:
        condition: service_healthy 
    image: amazon/aws-cli
    volumes:
      - "./schemas:/tmp/dynamo"
    environment:
      AWS_ACCESS_KEY_ID: 'FAKEID'
      AWS_SECRET_ACCESS_KEY: 'FAKEKEY'
      AWS_REGION: 'us-east-1'
    entrypoint:
      - bash
    command:
      '-c "for f in /tmp/dynamo/*.json; do aws dynamodb create-table --endpoint-url "http://dynamodb-local:8000" --cli-input-json file://"$${f#./}"; done"'
```

As you can see, even though I created AWS environments, I used fake credentials. If you're using a local environment, you don't need real credentials, but you do need to create the environments. Furthermore, I declared the dependencies with a condition; the dynamodb-local service must be initialized.

The main part of the code is the command, which is very simple. I read my volume with my descriptions and run the command `aws dynamodb create-table --endpoint-url "<http://dynamodb-local:8000>" --cli-input-json file://"$${f#./}"` for each file, where `$${f#./}` is the current file in the loop. The `#./` is necessary to replace `./` in the path and the `--endpoint-url` needs to point to the `dynamodb-local` service.

You now need to run the `docker compose up -d` command and wait for the execution to end.

![image](https://github.com/danielarrais/local-dynamodb-article/assets/28496479/bccfb384-807e-42c4-af50-d8ed4a59e794)

![image](https://github.com/danielarrais/local-dynamodb-article/assets/28496479/9a421af0-dc26-4032-ae1a-316f6e511203)

You can now create a local environment without needing to initialize your tables outside of Docker with other commands. You can even go a step further by creating another similar service to insert data into the tables using other `aws cli` commands.

# Links

1. [Git Repository](https://github.com/danielarrais/local-dynamodb-article)