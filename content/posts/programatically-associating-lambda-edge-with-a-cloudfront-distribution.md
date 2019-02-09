---
layout: blog
title: Programatically associating Lambda@Edge with a CloudFront distribution
date: '2018-04-26T12:10:31-04:00'
cover: /images/lambda-cf.png
categories:
  - AWS
  - CloudFront
  - Lambda@Edge
---
[Here's a little script ](https://github.com/marksteele/lambdaAtEdgeToCloudFront)which can be used to programatically associate Lambda@Edge functions with CloudFront.

![null](/images/lambda-cf.png)

<!--more-->



It's input is a JSON configuration file (`cf-associations.json`) that looks like this:

```json
[
  {
    "distributionId":"ASDFASDFASDF",
    "DefaultCacheBehavior": 
      {
        "Quantity": 2,
        "Items": [
          {
            "LambdaFunctionARN": "arn:aws:lambda:us-east-1:12345:function:yourfunctionname:",
            "EventType": "origin-response"
          },
          {
            "LambdaFunctionARN": "arn:aws:lambda:us-east-1:12345:function:baisc-auth:",
            "EventType": "viewer-request"
          }
        ]
      },
    "CacheBehaviors": [
      {
        "path": "/somepathcsp*",
        "rules": 
        {
          "Quantity": 1,
          "Items": [
            {
              "LambdaFunctionARN": "arn:aws:lambda:us-east-1:12345:function:anotherfunction:",
              "EventType": "origin-response"
            }
          ]
        }
      },
      {
        "path": "/someotherpath*",
        "rules": 
        {
          "Quantity": 0
        }
      }
    ]
  }
]
```

The rules are a 1 to 1 mapping to the JSON structure returned by the `aws cloudfront get-distribution-config` command, with the only difference being that the ARN does not include the version number. 

The script parses the config, grabs the current distribution configuration, publishes a new Lambda version (if necessary), and associates the configured behaviors to the right Lambda functions.
