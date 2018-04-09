---
title: "Bash Tips"
date: 2018-04-09T13:50:47-04:00
draft: false
---

Some odds and ends...

<!--more-->

# Invalidate CloudFront and wait

```bash
invalidate-cf-and-wait() {   id=$(aws cloudfront create-invalidation \
--distribution-id $1 --path '/*' --profile $2 | egrep Id | awk -F'"' '{ print $4}' ); \
echo "Waiting for invalidation $id";\
aws cloudfront wait invalidation-completed --id $id \
--distribution-id $1 --profile $2 && \
osascript -e "display notification 'Invalidation $id completed' with title 'CF Invalidation'" ; }
```