---
title: "unserializing php from pdi"
date: "2014-01-10"
slug: "2014/01/10/unserializing-php-from-pdi"
categories:
  - php
cover: /images/php.png
---


Here's a quick post that explains how to do something which may not be obvious.

The scenario: You've got some serialized data stored in a not-so-portable data interchange format (serialized PHP), 
and would like the data to be made available as part of a PDI transformation.

<!--more-->

Easy peasy lemon squeezy. Using the Javascript step, and the function found [here http://phpjs.org/functions/unserialize/].

Here's the transformation
![Transform](/images/pdi-php-js-1.png)

Next we'll create a data grid to test it out.

![Data grid 1](/images/pdi-php-js-2.png)

This is the data in our grid.

![Data grid 2](/images/pdi-php-js-3.png)

This is our transformation script.
![JS transform](/images/pdi-php-js-4.png)

In the javascript step, right click at the top of the code pane and add a new script. Then click on it. Right-click on the tab for that script, 
and select make start script. This will ensure that this code only gets executed once as opposed to once for every row (will make it faster).

You'll want to copy paste the javascript library linked above, and tweak it a little. First, change the function definition to resemble this screenshot:

![JS start](/images/pdi-php-js-5.png)

And at the end, add a semi-colon.

Running the transform should produce something like this:

![Result](/images/pdi-php-js-6.png)

