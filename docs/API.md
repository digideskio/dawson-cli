
# API Documentation

* [`package.json` fields reference](#packagejson-fields-reference)
* [`api.js` fields specification](#apijs-fields-specification)
  * [Customizing a dawson template](#customizing-a-dawson-template)
* [`Lambda parameters reference`](#lambda-parameters-reference)
  * [CloudFront default whitelisted headers](#cloudfront-default-whitelisted-headers)
* [`function property reference`](#function-property-reference)


---

## `package.json` fields reference

You must define a `dawson` property, as follows:

* **appName** (**required**, string): your app name, used in template and resource names. Keep it short but unique.  
  NOTE: changing this causes the whole application to be deployed from scratch.
* **domains** (**required**, list of strings): a list of at least one domain name to set as [CloudFront CNAME](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/CNAMEs.html). Domains must be unique globally in AWS.
* **zipIgnore** (list of strings): a list of partial paths to ignore when zipping lambdas. **Do not** ignore `node_modules`.
* **cloudfront** (boolean, defaults to `true`): if `false`, the default CloudFront distribution won't be added to the CloudFormation template, so:
  * if you are deploying a new app, the deploy will be very quick, and no distribution will be created
  * if you are updating an app that has been previously deployed with `cloudfront !== false`, the distribution will be **deleted** (this will take ~20min)
  * if you are referencing the distribution from a custom resource your stack will fail
 
 *This option controls the behaviour of the default CloudFront distribution that dawson creates, and does not apply to any custom resource.*
* **cloudfrontRootOrigin** (either `assets` or `api`, defaults to `api`):
  * if "assets", use S3 assets (uploaded via `$ dawson upload-assets`) as [Default Cache Behaviour](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/distribution-web-values-specify.html#DownloadDistValuesCacheBehavior), i.e.: serve the root directory from S3, useful for Single Page Apps. Requests starting with `/prod` are forwarded to API Gateway.
  * if "api", use your API as Default Cache Behaviour. Requests starting with `/assets` are forwarded to S3 assets bucket.

##### Example
```js
"dawson": {
  "appName": "myapp", // required, unique
  "domains": [
    "mydomain123.example.com" // required, unique
  ],
  "zipIgnore": [
    "frontend"
  ],
  "cloudfront": true,
  "cloudfrontRootOrigin": "api"
},
```


## `api.js` file specification

You must have an `api.js` file in the folder where you will run `dawson`. This file may use ES2017 and must, at least, export a function.


#### Customizing a dawson template

If you need to add more Resources or modify Resources and Outputs created by `dawson` you may export a `processCFTemplate` function from your `api.js`.  
This function takes a CloudFormation Template (object, like [this](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/quickref-lambda.html)) and must return the new (possibly updated) template. `processCFTemplate` will be invoked right before calling CloudFormation's `UpdateStack` API. `dawson` will not process or parse this template further.

CloudFormation is very powerful, but sometimes it might be very complex. Keep in mind that CloudFormation [Template Reference](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-reference.html) is your friend.

##### Example 1: Adding a DynamoDB Table

```js
const PROJECTS_TABLE = {
  Resources: {
    ProjectsTable: {
      "Type": "AWS::DynamoDB::Table",
      "Properties": {
        "AttributeDefinitions": [{
          "AttributeName": "ProjectId",
          "AttributeType": "S"   
        }],
        "KeySchema": [{
          "AttributeName" : "ProjectId",
          "KeyType" : "HASH"
        }],
        "ProvisionedThroughput" : {
          "ReadCapacityUnits" : "1",
          "WriteCapacityUnits" : "1"
        }
      }
    }
  },
  Outputs: {
    ProjectsTable: {
      "Value": { "Ref": "ProjectsTable" }
    }
  }
};

export function processCFTemplate(template) {
  return {
    Resources: {
      ...template.Resources,
      ...PROJECTS_TABLE.Resources
    },
    Outputs: {
      ...template.Outputs,
      ...PROJECTS_TABLE.Outputs
    }
  };
}
```

##### Example 2: Modifying a Template

Simply merge the new properties in the template.  
You may also use `Object.assign` or any other library to make this look nicer.

```js
export function processCFTemplate(template) {
  return {
    ...template,
    Resources: {
      ...template.Resources,
      myAppAssets: {
        ...template.Resources.myAppAssets,
        Properties: {
          ...template.Resources.MyAppAssets.Properties,
          NotificationConfiguration: {
            LambdaConfigurations: [{
              Event: "s3:ObjectCreated:*",
              Filter: {
                S3Key: { Rules: [{
                  Name: "prefix", Value: "uploads/",
                  Name: "suffix", Value: ".doc"
                }] }
              }
              "Function": { "Fn::GetAtt": ["convertToPDF", "Arn"] }
              // in order for s3 to invoke Lambda you should also add the correct AWS::Lambda::Permission
            }]
          }
        }
      }
    }
  };
}
```

Please, do not forget to return the **whole** template object, and not just the new Resources.



## Lambda parameters reference

Unless `api.noWrap` is `true`, your function will be called with a single argument, which will follow
this spec:

```js
{
  "params": {
    "querystring": {}, // parameters from the querystring
    "path": {}, // path parmeters, captured by `{}` in function `path`s
    "header": {} // some HTTP headers from the client, see below
  },
   // body: the request body (useful only for POST and PUT requests)
   //  currently, only application/json bodies will be parsed
  "body": $input.json('$'),
  "meta": {
    // expectedResponseContentType: the content-type that is expected to be returned
    //  by the function. This is used internally to wrap the returned value
    //  for the API Gateway Method Response.
    "expectedResponseContentType": "<text/html|application/json>"
  },
  "stageVariables" : {
    // this will include all CloudFormation Template Outputs, as listed
    //  by `$ dawson describe`
  }
}
```

#### CloudFront default whitelisted headers

By default we set CloudFront to only [forward](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/forward-custom-headers.html) these HTTP Headers:
* `Authorization`
* `Accept`
* `Content-Type`
* `Origin`
* `Referer`
* `Access-Control-Request-Headers`
* `Access-Control-Request-Method`

This applies only if you are using the CloudFront distribution endpoint and does not apply if
you are invoking API Gateway directly or via a custom proxy.
You may add or modify whitelisted headers, see [Customizing a dawson template](#customizing-a-dawson-template).


*Internally, `dawson` uses a [Passthrough Parameter-Mapping Template](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-mapping-template-reference.html).*


## Function property reference

Each function exported by the top-level `api.js` must have an `api` property.

* **path** (*required*, string): HTTP path to this functions. Must be unique in your whole app. You may use path parameters placeholder, as in API Gateway, by sorrounding a parameter name with `{}`. Do **not** include leading/trailing slashes.
* **method** (string, defaults to GET): HTTP method.
* **responseContentType** (string, defaults to `text/html`): Content-Type to set in API Gateway's response. Valid values are: `application/json`.
* **policyStatements** (list of maps): Policy statements for this Lambda's Role, as you would define in a [CloudFormation template](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-iam-policy.html#cfn-iam-policies-policydocument).
* **noWrap** (boolean, defaults to `false`): If true, this function call won't be wrapped in a Promise and it will be directly exported as the lambda's handler. It will receive these arguments (may vary based on the runtime): `event`, `context`, `callback`. For `application/json` content type, you *must* invoke the callback passing your stringified response in a `response` property (e.g.: `callback(null, { response: '"wow"' })`. For `text/html` content type: `callback(null, { html: '<html>....' })`.
* **runtime** (string, defaults to `nodejs4.3`): Lambda runtime to use. Only NodeJS runtimes make sense. Valid values are `nodejs` and `nodejs4.3`. You should only use the default runtime.


##### Example
```javascript
export function myFunction() { ... });
// or: export async function myFunction() { ... await ... });

myFunction.api = {
  path: 'message/{messageId}', // required
  method: 'GET',
  responseContentType: 'text/html',
  policyStatements: [{
    Effect: "Allow",
    Action: ["dynamodb:Query", "dynamodb:GetItem"],
    Resource: { "Fn::Join": ["", [
      "arn:aws:dynamodb",
      ":", { "Ref" : "AWS::Region" },
      ":", { "Ref" : "AWS::AccountId" },
      ":", "table/", { "Ref": "UsersTable" },
      "*",
    ]] },
  }],
  noWrap: false,
  runtime: "nodejs4.3"
};
```