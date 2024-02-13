---
published: false
title: 'Migrating from API Gateway V2 to V1 to get REST API features'
cover_image: ''
description: 'The guide to move from AWS HTTP API to REST API, with a focus on Lambda integration with serverless framework.'
tags: aws, apigateway, serverless, lambda
series:
canonical_url:
---

This guide aims to help you migrate a serverless or CloudFormation stack with API Gateway v2 (HTTP API) integrated with lambdas to the same stack using API Gateway v1 (REST API), with a focus on:

- Handling API paths constraints
- Input and Output serialization (and in particular these breaking changes: JWT authentication, CORS)

AWS REST API has a lot of features that HTTP API do not benefit from: cache at edge, WAF integration, API keys. Even if some features can be circumvented, like using CloudFront for cache at edge, some are not and a migration can be necessary. For a detailed comparison between the two versions, see this guide: <https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-vs-rest.html>

**💡 TLDR**: have a look to this [repo](https://github.com/guiyom-e/sls-apigateway-v2-to-v1) for Typescript types, helpers to format API responses and a serverless template with the use of both API versions.

## 🗜️ API path constraints are different

### In REST API, each part of an API path is a resource

In REST API, paths must be defined once, because each path part is a resource. This is not an issue with HTTP API.

If you need to deploy `/actions/list` and `/actions/toggle`, you need to deploy 3 [ApiGateway Resource](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-apigateway-resource.html), `/action`, `/list` and `/toggle`, one for each path.
If you define `/action` twice, you will get `409 Conflict "Another resource with the same parent already has this name: actions"`

With Serverless framework, it is done for you, except if you deploy different services with a path in common: in this case, you have no choice but to rename it.

### Variables in paths are managed differently

Let’s say you have defined two routes `/network/{networkId}/actions/list` and `/network/{userId}/profile` in the HTTP API. Even if we can challenge this api path structure, it is possible with HTTP API… but not with REST API, because only one variable path part name is allowed.

In CloudFormation, you should encounter this error: `"A sibling ({networkId}) of this resource already has a variable path part -- only one is allowed"`.

Two options: either rename `userId` to `networkId` (if it makes sense 🤔) or change the path!

## ➡️ Input is different

There are two versions of the input payload, [Payload 1.0 and Payload 2.0](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-develop-integrations-lambda.html). For Typescript users, I wrote the [payload types](https://github.com/guiyom-e/sls-apigateway-v2-to-v1/blob/main/src/apiPayload.ts), as they are not exposed in AWS SDK.

### Handling headers and query strings with multiple values

With API Gateway v2, query strings and headers can have multiple values for the same key. In this case, values are comma-separated in the input object `headers` or `queryStringParameters`.

```js
// Payload 1.0 (REST API)
{ 
  ...
  "headers": { "singleValueParam": "value1" },
  "multiValueHeaders": { "multiValueParam": ["value1", "value2"] }
}

// Payload 2.0 (HTTP API)
{ 
  ...
  "headers": { "singleValueParam": "value1", "multiValueParam": "value1,value2" }
}
```

### Authentication is different... and Cognito groups are formatted differently!

If you use a Cognito authorizer, you can find similar keys in both payloads... but they are not the same!

For instance for a user in Cognito groups "group1" and "group2":

```js
// Payload 1.0 (REST API)
{ 
  ...
  "requestContext": { 
    "authorizer": {
      "claims": {
        "cognito:groups": "group1,group2",
        "cognito:username": "bob@example.com",
        "email": "bob@example.com"
      }
    }
   }
}

// Payload 2.0 (HTTP API)
{ 
  ...
  "requestContext": { 
    "authorizer": {
      "jwt": {
        "claims": {
          "cognito:groups": "[group1 group2]",
          "cognito:username": "bob@example.com",
          "email": "bob@example.com"
        }
      }
    }
   }
}
```

Therefore, Cognito groups must be parsed differently!

### Other parameters are not at the same place...

.. but I won't go through other differences, but they are a lot that can be easily found in [these types](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-develop-integrations-lambda.html).

## 🔀 In REST API, CORS are handled at method level

In HTTP API, a global configuration can be set ; in REST API it is at method level.
So when moving to REST API, you need to edit every method to add CORS for preflight requests. And for other requests, you need to update the output of the lambda manually (see [next section](#output-is-different-even-if-a-lambda-integration-is-chosen)).

## ⬇️ Output is different, even if a Lambda integration is chosen

With HTTP API, if no `statusCode` key is defined, the output payload is automatically stringified to a response body in a HTTP response with  status 200 (see [documentation](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-develop-integrations-lambda.html#http-api-develop-integrations-lambda.v2))
. This is not the case in REST API, so you need to format your output manually. For an easier migration, you can set up a middleware based on this [code](https://github.com/guiyom-e/sls-apigateway-v2-to-v1/blob/main/src/apiResponseFormatter.ts), or if you use `@middy` middleware, based on this [code](https://github.com/guiyom-e/sls-apigateway-v2-to-v1/blob/main/src/apiResponseFormatterMiddleware.ts).
These helpers also add CORS headers, as this is not handled by REST API.

## Finally, should I move to REST API?

In short, the REST API comes with more features, but is more complex whereas HTTP API is a [three-times cheaper](https://aws.amazon.com/api-gateway/pricing/) and turnkey solution, especially if you use a JWT authentication and needs CORS headers. Moving from one API to another is clearly possible, but painful for a big application with multiple services sharing the same API Gateway.
