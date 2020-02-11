---
layout: blog
title: Parsing SFTP logs with Cloudwatch log Insights
date: '2020-02-11T10:32:18-05:00'
cover: /images/aws.png
categories:
  - aws
  - security
---
This is a bit painful, but here it is:

```
filter @message like /OPEN/
| parse @message /^(?<logstream>\S+)\s(?<Action>\S+)(\s(SourceIP=(?<SourceIP>\S+)\sUser=(?<User>\S+)\sHomeDir=(?<HomeDir>\S+)\sClient=(?<Client>("[^"]+?"|\S+))\sRole=.*|Path=(?<Path>\S+)\s((BytesIn=(?<BytesIn>\S+)|BytesOut=(?<BytesOut>\S+)|Mode=(?<Mode>\S+))|NewPath=(?<NewPath>\S+))))?$/
| sort @timestamp asc
| display @timestamp, logstream, Action, SourceIP, User, HomeDir, Client, Path, BytesIn, Mode,BytesOut, NewPath
```

You can remove the filter line in order to get all types of commands, or change the filter to narrow down your results.
