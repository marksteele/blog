+++
title = "calculating distances between coordinates in sql"
date = "2012-08-22"
slug = "2012/08/22/calculating-distances-between-coordinates-in-sql"
Categories = []
+++

The following stored functions can calculate the distance between two coordinates using a couple different approximation methods. The return value is in miles.

Haversine approximation
<!--more-->
``` sql Haversine
DELIMITER $$
DROP FUNCTION IF EXISTS `mydb`.`distance_HAVERSINE` $$
CREATE FUNCTION `distance_HAVERSINE`(
  lat1 FLOAT(9,6), 
  lon1 FLOAT(9,6),
  lat2 FLOAT(9,6), 
  lon2 FLOAT(9,6)
) 
RETURNS INT
DETERMINISTIC
COMMENT 'distance calculated using haversine function'
BEGIN
RETURN 3956 * 2 *
ASIN(
  SQRT(
    POWER(SIN((lat1 - lat2) * pi()/180 / 2), 2) + 
    COS(lat1 * pi()/180) * 
    COS(lat2 * pi()/180) * 
    POWER(SIN((lon1 - lon2) * pi()/180 / 2), 2)
  )
);
END $$
DELIMITER ;
```

Spherical cosines

``` sql
DELIMITER $$
DROP FUNCTION IF EXISTS `mydb`.`distance_COSINES` $$
CREATE FUNCTION `distance_COSINES`(
DELIMITER $$
DROP FUNCTION IF EXISTS `mydb`.`distance_COSINES` $$
CREATE FUNCTION `distance_COSINES`(
  lat1 FLOAT(9,6), 
  lon1 FLOAT(9,6),
  lat2 FLOAT(9,6), 
  lon2 FLOAT(9,6)) 
RETURNS INT
DETERMINISTIC
COMMENT 'distance calculated using spherical law of cosines function'
BEGIN
RETURN 3956 *
ACOS(
SIN(lat1 * 0.0174532925) * 
SIN(lat2 * 0.0174532925) +
COS(lat1 * 0.0174532925) * 
COS(lat2 * 0.0174532925) *
COS((lon2 - lon1) * 0.0174532925)
);
END $$
DELIMITER ;
```


Furthermore this can be sped up significantly by making a few assumptions.

1 degree of Latitude equals approximately 69 miles
1 degree of Longitude equals approximately ABS(COS(radian(origin point latitude in degrees)))*69 miles.

Here's a sample of how we can use these assumptions. Assuming we have a table of zip codes with logitutde 
and latitude, we can retrieve all zip codes within a range from a starting zip code. We can thus figure out 
all zip codes within 5 miles of 90210 with a query like:

``` sql
SELECT
      dest.col_zip_code
    FROM
      tbl_zip_code dest,
      tbl_zip_code orig
    WHERE
      orig.col_zip_code = '90210' AND
      dest.col_latitude BETWEEN
        orig.col_latitude - 5/69 AND
        orig.col_latitude + 5/69 AND
      dest.col_longitude BETWEEN
        orig.col_longitude - 5/abs(cos(radians(orig.col_latitude))*69) AND
        orig.col_longitude + 5/abs(cos(radians(orig.col_latitude))*69);
```

This approach is several orders faster than the previous two functions with similar accuracy.
