---
layout: blog
title: String set manipulation functions for MySQL
date: '2018-10-11T10:10:05-04:00'
cover: /images/mysql.png
categories:
  - MySQL
---
While I really don't advise using comma-separated values inside text columns, sometimes you inherit things...

Here's a way to check for set inclusion and how to manipulate those sets using SQL functions.

```
drop function if exists IN_SET;
drop function if exists ADD_TO_SET;
drop function if exists REMOVE_FROM_SET;

DELIMITER //
CREATE FUNCTION IN_SET(
  $str     TEXT,
  $strlist TEXT
)
RETURNS BOOL
DETERMINISTIC
BEGIN
  DECLARE inset INT DEFAULT 0;
  SET inset = (SELECT FIND_IN_SET($str, $strlist));
  RETURN (!ISNULL(inset) AND inset > 0);
END
//

CREATE FUNCTION ADD_TO_SET(
  $str     TEXT,
  $strlist TEXT
)
RETURNS TEXT
DETERMINISTIC
BEGIN
  IF (IN_SET($str, $strlist))
  THEN
    RETURN $strlist;
  END IF;
  RETURN IF($strlist = '', $str, CONCAT_WS(',', $strlist, $str));
END
//

-- This will catch a some duplicated entries in the set, but there 
-- are edge cases. 
-- Functions aren't allowed to be recursive in MySQL.
-- This could be re-implemented as a stored proc with recursion...
CREATE FUNCTION REMOVE_FROM_SET(
  $str     TEXT,
  $strlist TEXT
)
RETURNS TEXT
DETERMINISTIC
BEGIN
  -- Entire set is the target
  IF ($strlist = $str) THEN
        RETURN '';
  END IF;
  SET @str_size = length($str);
  IF (IN_SET($str, $strlist))
  THEN
    SET @RET = $strlist;
    -- In the middle
    IF (@RET LIKE CONCAT('%,',$str,',%')) THEN
        SET @RET = CONCAT(SUBSTRING(@RET,1,LOCATE(CONCAT(',',$str,','),@RET)),SUBSTRING(@RET,LOCATE(CONCAT(',',$str,','),@RET)+@str_size+2));
    END IF;
    -- Leading
    IF (SUBSTRING(@RET,1,@str_size) = $str) THEN
        SET @RET = SUBSTRING(@RET, @str_size+2);
    END IF;
    -- Trailing
    IF (@RET LIKE CONCAT('%,',$str)) THEN
        SET @RET = SUBSTRING(@RET,1,length(@RET) - (@str_size+1));
    END IF;
    RETURN @RET;
   ELSE
      RETURN $strlist;
   END IF;
END
//

DELIMITER ;
```
