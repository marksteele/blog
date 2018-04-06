+++
title = "sample esp queries"
date = "2011-10-25"
slug = "2011/10/25/sample-esp-queries"
Categories = []
+++

Emit when something hasn't been seen in a while:

```
select 
  * 
from 
  NoitMetricNumeric.std:groupwin(uuid,name).win:time(5 minutes).std:lastevent().std:size() 
where 
  size = 0 group by uuid, name
```
<!--more-->
Threshold change data stream population:

```
insert into 
  ThresholdChange 
select 
  *, 
  case 
    when value > 0.000821 
      then 'bad' 
      else 'good' end 
    as status from 
NoitMetricNumeric(uuid='1b4e28ba-2fa1-11d2-883f-b9b761bde3fb',name='average')
```

Emitting on change:

```
select 
  tc.name as metric,
  tc.uuid as uuid,
  tc.noit as noit,
  tc.time as time,
  tc.value as value,
  tc.status as status,
  cds.name as check,
  cds.module as 
  module,cds.prefix as prefix,
  cds.target as target 
from 
  ThresholdChange.std:groupwin(uuid,name).win:length(2) tc, 
  CheckDetails cds 
where 
  cds.uuid = tc.uuid and tc.status <> prev(tc.status)
```
