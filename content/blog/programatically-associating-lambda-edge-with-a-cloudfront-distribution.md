---
layout: blog
title: Programatically associating Lambda@Edge with a CloudFront distribution
date: '2018-04-26T12:10:31-04:00'
thumbnail: /images/lambda-cf.png
categories:
  - AWS
  - CloudFront
  - Lambda@Edge
---
Here's a little script which can be used to programatically associate Lambda@Edge functions with CloudFront.

![](/images/lambda-cf.png)

<!--more-->

```perl
#!/usr/bin/perl
use JSON;

my $CONFIG;
{
local $/ = undef;
open(F,"./cf-associations.json") || die $!;
$CONFIG = decode_json(<F>);
close(F);
}

foreach my $distribution (@{$CONFIG}) {
  unlink("./$distribution->{'distributionId'}.json");
  my $CFConfig = decode_json(`aws cloudfront get-distribution-config --id $distribution->{'distributionId'}`);
  if ($distribution->{'DefaultCacheBehavior'}) {
    foreach my $cb (@{$distribution->{'DefaultCacheBehavior'}->{'Items'}}) {
      $cb->{'LambdaFunctionARN'} =~ /^.+:(.+):$/;
      my $newArn = `aws lambda publish-version --function-name $1 --region us-east-1 | jq -r '.FunctionArn'`;
      chomp($newArn);
      $cb->{'LambdaFunctionARN'} = $newArn;
    }
    $CFConfig->{'DistributionConfig'}->{'DefaultCacheBehavior'}->{'LambdaFunctionAssociations'} = $distribution->{'DefaultCacheBehavior'};
  } 
  foreach my $cache (@{$distribution->{'CacheBehaviors'}}) {
    foreach my $cb (@{$cache->{'rules'}->{'Items'}}) {
      $cb->{'LambdaFunctionARN'} =~ /^.+:(.+):$/;
      my $newArn = `aws lambda publish-version --function-name $1 --region us-east-1 | jq -r '.FunctionArn'`;
      chomp($newArn);
      $cb->{'LambdaFunctionARN'} = $newArn;
    }
    foreach my $item (@{$CFConfig->{'DistributionConfig'}->{'CacheBehaviors'}->{'Items'}}) {
      if ($item->{'PathPattern'} eq $cache->{'path'}) {
        $item->{'LambdaFunctionAssociations'} = $cache->{'rules'};
        last;
      }
    } 
  
  }
  open(F,">./$distribution->{'distributionId'}.json");
  print F encode_json($CFConfig->{'DistributionConfig'});
  close(F);
  print `aws cloudfront update-distribution --id $distribution->{'distributionId'} --region us-east-1 --distribution-config=file://$distribution->{'distributionId'}.json --if-match $CFConfig->{'ETag'}`;
  unlink("./$distribution->{'distributionId'}.json");
}
```

It's input is a JSON configuration file (`cf-associations.json`) that looks like this:

```
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
