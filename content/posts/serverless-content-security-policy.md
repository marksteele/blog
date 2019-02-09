---
layout: blog
title: Serverless content security policy
date: '2018-04-27T11:40:02-04:00'
cover: /images/csp_shield_logo.jpg
categories:
  - AWS
  - Content Security Policy
  - Lambda
  - Lambda@Edge
  - CloudFront
  - serverless
---
![](/images/csp_shield_logo.jpg)

Add content security policy (CSP) to your site without changing your backends (or when you don't have backends and are using static site origins). Here's how!

<!--more-->

For an overview of why you should add content security policy to your site, please read [here](https://developers.google.com/web/fundamentals/security/csp/), [here](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP), and [here](https://csp.withgoogle.com/docs/why-csp.html).

As it's sometimes tricky to get the back-end updated to add additional headers, I decided to leverage Lambda@Edge to serve the HTTP headers from the CloudFront edge locations.

Go grab the code [here](https://github.com/marksteele/serverless-csp), follow the fine instructions, and you should be good to go!
