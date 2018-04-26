---
layout: blog
title: Invalidate CloudFront with Lambda
date: '2018-04-26T13:48:12-04:00'
thumbnail: /images/lambda.png
categories:
  - CloudFront
  - AWS Lambda
  - API Gateway
---
I've written a little Serverless app that uses API Gateway and Lambda to expose an API to invalidate a CloudFront distribution.

Handy to have something like this in a build pipeline.

[Code here](https://github.com/marksteele/serverless-invalidate-cf)
