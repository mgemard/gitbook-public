---
description: 'Notes for https://serverless.com'
---

# Serverless

## Commands

### Update serverless

```bash
npm i -g serverless
```

### Create typescript project

```bash
serverless create \
  --template aws-nodejs-typescript \
  --path aws-lambdas
```

## Best Practices / Learnings So Far

* Don't overload your `serverless.yml` with many services.
  * See DynamoDB gotcha
* Failures can leave your deployment in an uncertain state
  * You may have to delete the file from S3
* Rename your Amazon API Gateway
  * [https://github.com/aizatto/serverless-prototypes/tree/master/aws-apigateway-proxy](https://github.com/aizatto/serverless-prototypes/tree/master/aws-apigateway-proxy)
* Errors with your cloudformation can really put you in an inconsistent state
* Cannot update more than one GSI at a time
* Use `tags` in your `serverless.yml` 
  * You'll have to manually tag your IAM roles. AWS doesn't currently support cloudformation tagging of IAM roles.
* Use `sls remove` to remove CloudFormation stacks

### Recommended default `serverless.yml`

[https://serverless.com/framework/docs/providers/aws/guide/serverless.yml/](https://serverless.com/framework/docs/providers/aws/guide/serverless.yml/)

Naming goals:

```bash
$project-$stage-$service
```

For example:

```bash
url-shortener-prod-lambdas
url-shortener-prod-dynamodb
url-shortener-dev-lambdas
url-shortener-dev-dynamodb
```

{% code title="serverless.yml" %}
```yaml
service:
  name: naming-lambdas
  awsService: lambda
  awsName: naming-${opt:stage} 

provider:
  memorySize: 128
  timeout: 30 # API Gateway limits to 30 seconds
  apiName: ${self:service.awsName}
  tags:
    product: ${self:service.awsName}
  deploymentBucket:
    tags:
      product: ${self:service.awsName}
  stackName: ${self:service.awsName}-${self:service.awsService}
  stackTags:
    product: ${self:service.awsName}

resources:
  Resources:
    ApiGatewayRestApi:
      Type: AWS::ApiGateway::RestApi
      Properties:
        Name: ${self:service}-${opt:stage}
        
functions:
  hello:
    handler: handler.hello
    name: ${self:provider.stackName}-init
```
{% endcode %}

### One or many \`serverless.yml\`

When working on a project, should you use one or many `serverless.yml` files?

* For demos, its easier to just use one.
* For production use many.

#### One serverless.yml

Pros:

* Great for demonstrating a single concept

Cons:

* Hard to isolate services when needing to upgrade

Great for demoing a serverless project.

#### Many serverless.yml

Pros:

* Distribute your services across multiple `serverless.yml`
* Easier to limit deployments

Cons:

* Requires more configuration

Necessary when you need to deploy across different regions. For example: `ap-southeast-1` and `us-esat-1,` this is needed because SES only works in certain regions.

## AWS - Invoke Local

[https://serverless.com/framework/docs/providers/aws/cli-reference/invoke-local/](https://serverless.com/framework/docs/providers/aws/cli-reference/invoke-local/)

Great for testing functions locally.

```bash
serverless invoke local --function functionName
```

If you want to pass a url into the `event.body`

```javascript
JSON.stringify({ body: JSON.stringify($value)})
```

Then:

```bash
serverless invoke local --function functionName --data $string
```



## API Gateway Gotchas

### API Gateway Bad Naming

By default Serverless names the API Gateway as `$stage-$service`

This is not scaleable when you have many services. To fix this update your `serverless.yml`:

```yaml
provider:
  apiName: ${self:service}-${stage}
```

## DynamoDB

Dynamic table names

{% code title="serverless.yml" %}
```yaml
provider:
  environment:
    COUNTERS_TABLE: ${self:custom.DYNAMODB.COUNTERS_TABLE}
    URLS_TABLE: ${self:custom.DYNAMODB.URLS_TABLE}

custom:
  DYNAMODB: ${file(./tables.js)}
```
{% endcode %}

{% code title="tables.js" %}
```javascript
module.exports = (serverless) => {
  const tables = {
    COUNTERS_TABLE: 'counters',
    URLS_TABLE: 'counters',
  };

  if (serverless.pluginManager.cliCommands[0] !== 'deploy') {
    return tables;
  }

  const stage = serverless.pluginManager.cliOptions['stage'];

  // need to test if I don't use the provider stage
  const PREFIX = `prototype-messenger-bot-${stage}`;

  const deployed_tables = {};
  for (const key in tables) {
    const table = tables[key];
    deployed_tables[key] = `${PREFIX}-${table}`
  }

  return deployed_tables;
}
```
{% endcode %}

### Gotchas

WARNING: Changing table names drops the original database.

Be very careful.

* Cannot rename tables \(also a dynamodb restriction\)
* Cannot change indices
  * I wanted to created a new index from `end_time` to `start_time`  but it told me to create a new index
* Changing/adding/remove indexes is painful

#### Cannot reuse existing DynamoDBs

Serverless cannot reuse existing DyanmoDB tables. It will error our if you try to use it:

> An error occurred: eventsTable - $table already exists.

[https://github.com/serverless/serverless/issues/3183](https://github.com/serverless/serverless/issues/3183)

#### Billing Mode: Pay Per Request

I ran into bugs trying to move to "PAY\_PER\_REQUEST" mode, and then I couldn't deploy, and when I think I fixed this, I got this error:

> An error occurred: $table - Subscriber limit exceeded: Update to PayPerRequest mode are limited to once in 1 day\(s\). Last update at Sat Feb 23 03:07:45 UTC 2019. Next update can be made at Sun Feb 24 03:07:45 UTC 2019 \(Service: AmazonDynamoDBv2; Status Code: 400; Error Code: LimitExceededException; Request ID: E2GS9JLCCLIUH8BNIPQN32UEDBVV4KQNSO5AEMVJF66Q9ASUAAJG\).

### Plugin: serverless-dynamodb-local

[https://www.npmjs.com/package/serverless-dynamodb-local](https://www.npmjs.com/package/serverless-dynamodb-local)

This seems like a poor plugin.

* Have to use `v0.2.30`
* Constantly has problems running
  * Have to consistently remove and install

Useful locally for:

* migrations
* seeding

Use `v0.2.30`, there are problems with the latest version.

```bash
yarn add --dev serverless-dynamodb-local@0.2.30
```

## Roles

By default, the roles you create apply to all functions.

You can define individual roles to functions via [https://github.com/functionalone/serverless-iam-roles-per-function](https://github.com/functionalone/serverless-iam-roles-per-function)

## Serverless Alternatives

* [https://up.docs.apex.sh/](https://up.docs.apex.sh/)
* [https://zeit.co/](https://zeit.co/)
* [https://github.com/awslabs/serverless-application-model](https://github.com/awslabs/serverless-application-model)
* [https://www.netlify.com/docs/functions/](https://www.netlify.com/docs/functions/)
* [https://workers.dev/](https://workers.dev/)
* [http://fnproject.io/](http://fnproject.io/)

### AWS Serverless Application Model \(SAM\)

Pros:

* Directly define CloudFormation naming

Cons:

* More complex flags to use for deploying
* No templating

## Personal Prototypes / Toys / Experiments

* [URL Shortener](https://github.com/aizatto/url-shortener/)
* [build.my](https://github.com/aizatto/build.my)
* [https://github.com/aizatto/serverless-prototypes](https://github.com/aizatto/serverless-prototypes)



