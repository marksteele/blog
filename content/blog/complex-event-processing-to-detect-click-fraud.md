+++
title = "complex event processing to detect click fraud"
date = "2013-10-19"
slug = "2013/10/19/complex-event-processing-to-detect-click-fraud"
Categories = []
+++

Here's another use-case for CEP: Detecting uniqueness over time. A use-case for this type of pattern is identifying click fraud.

Once more, to see how to get everything up and running, see my previous posts.

In our fictitious scenario, we're going to assume we want to see a stream of incoming data filtered to only output unique data given a subset of uniqueness over a 24 hour period.
<!--more-->

new-hope/config/hope.xml:

```xml 
<?xml version="1.0" encoding="UTF-8"?>
<esper-configuration
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xmlns="http://www.espertech.com/schema/esper"
xsi:noNamespaceSchemaLocation="esper-configuration-4-0.xsd">

  <event-type name="click">
    <java-util-map>
      <map-property name="affiliateid" class="int"/>
      <map-property name="promocodeid" class="int"/>
      <map-property name="siteid" class="int"/>
      <map-property name="ip" class="string"/>
      <map-property name="keyword" class="string"/>
      <map-property name="referrer" class="string"/>
      <map-property name="timestamp" class="int"/>
    </java-util-map>
  </event-type>
</esper-configuration>
```

new-hope/config/hope.epl: 

```sql
@Name('uniquecheckwin-silent')
CREATE WINDOW
  uniquecheck.std:unique(affiliateid,promocodeid,siteid,ip,keyword).win:time(24 hour) as
  (affiliateid int, promocodeid int, siteid int, ip string, keyword string);

@Name('clickmerge-silent')
ON click c
MERGE uniquecheck uc
WHERE
  uc.affiliateid = c.affiliateid AND
  uc.promocodeid = c.promocodeid AND
  uc.siteid = c.siteid AND
  uc.ip = c.ip AND
  uc.keyword = c.keyword
WHEN NOT MATCHED
  THEN
    INSERT(affiliateid,promocodeid,siteid,ip,keyword)
     SELECT affiliateid,promocodeid,siteid,ip,keyword
  THEN
    INSERT INTO uniqueclicks SELECT *;

@Name('uniques')
SELECT * FROM uniqueclicks;
```

This code will filter out items in the incoming stream that are unique on affilaiteid,promocodeid,siteid,ip, and keyword over a 24 hour period.

Not sure how well this will perform if you.re seeing millions of unique clicks per day, I may prototype this further and post benchmarks. Other equally valid approaches to solve this problem would include the use of bloom filters or even better continuous bloom filters.

Another potential issue here is that this assumes your log stream is occurring in real-time, so no ability to replay log data. In Esper, there is however a work-around. It is possible to disable Esper'ss time-keeping capabilities and manually feed it time increments. 
