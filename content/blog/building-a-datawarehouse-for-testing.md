+++
title = "building a datawarehouse for testing"
date = "2014-01-04"
slug = "2014/01/04/building-a-datawarehouse-for-testing"
Categories = []
+++

Overview
--------

A common problem when starting a new project is getting fixtures in place to facilitate testing of reporting functionality and refining data models. To ease this, I've created a PDI job that creates the dimension tables, and populates a fact table. 

<!--more-->

My inspiration is from a variety of sources (for example the CDG project from Webdetails).

Here is the job that controls everything. It doesn't do anything special except link the transformations to each other. 

![Data warehouse generator job](/images/JOB-GenDW.PNG)

Next up, the dimension data generator transformation.

![Dimension data generator transformation](/images/Transform-GenDimData.PNG)

This transformation creates a template JSON document which is parsed and used in other steps, as well as defines the size of the date dimension and the start date. Here is the step by step breakdown.

To start, we generate a dummy row to kick off the transformation.

![gendimdata-generate-rows](/images/Transform-GenDimData-step-generate-rows.PNG)

The following step is where all the dimensions are defined (might need to zoom this image a bit to see the text). We're defining a JSON structure in here that contains all the data we'll need later on for both populating the fact table as well as creating the dimensions.

![gen data template](/images/Transform-GenDimData-step-gen-data-template.PNG)

Once the data is serialized, the remaining steps configures the start date and number of days, then passes this information along to the next transformation.

![set constants](/images/Transform-GenDimData-step-add-constants.PNG)

Next up, we've got the dimension populator transformation.

![Populate dimensions](/images/Transform-PopDims.PNG)

Grab the input rows from previous transformation.

![Grab input](/images/Transform-PopDims-step-get-input.PNG)

Here is what the dimension generator looks like. As you can see, there's one of these steps for each dimension. In the example I've posted, I'm creating a dimension that contains a hierarchy with several levels.

![Dimension data generator](/images/Transform-PopDims-step-generate-dim.PNG)

Then we save the dimension data to the database

![Dim table output](/images/Transform-PopDims-step-table-output.PNG)

For the date and time dimensions, we do things slightly differently. To generate all the dates in the range requested, we kick off a javascript step that generates empty rows, one for each day requested.

![Date row gen](/images/Transform-PopDims-step-date-row-gen.PNG)

We create a sequence, which is going to be our technical key later on. It represents the number of days since the first day generated.

![Date sequence](/images/Transform-PopDims-step-sequence.PNG)

We next do some date math to enrich our working set a bit. In my date dimension, I'm actually not using all of these, but it's handy to have them around.

![Date calc](/images/Transform-PopDims-step-date-calc.PNG)

Finally we output to the dimension table.

![Date table output](/images/Transform-PopDims-step-table-output-date.PNG)

Similarly for time, we generate one row for each hour of the day.

![Time generator](/images/Transform-PopDims-step-time-gen.PNG)

And save it to the dimension table.

![Time table output](/images/Transform-PopDims-step-time-table-out.PNG)

Finally, we have the transformation to populate the fact table.

![FactGen](/images/Transform-fact-table.PNG)

Grab the input rows from the previous transformation.

![Grab input rows](/images/Transform-fact-table-get-input.PNG)

Generate a series of days in the range requested.

![Generate day rows](/images/Transform-fact-table-step-gen-days.PNG)

Create a sequence.

![Generate day sequence](/images/Transform-fact-table-step-sequence-days.PNG)

Do some date math.

![Date calc](/images/Transform-fact-table-step-date-calc.PNG)

Metadata cleanups

![Select values](/images/Transform-fact-table-step-select.PNG)

![Select values other tab](/images/Transform-fact-table-step-select-2.PNG)

Generate row data for each dimension. So I've chosen to generate data for every hour of every day in the requested date range as well as for one of the dimensions (banner in this case).

![Fact generator](/images/Transform-fact-table-step-fact-generator.PNG)

Here's the second (start) javascript script on that step.

![Fact generator tab 2](/images/Transform-fact-table-step-fact-generator-2.PNG)

Closing thoughts
----------------

The above approach obviously requires a bit of copy pasta and customization to get it working for a particular use case.

I would have liked to completely automate the creation of dimensions and data generation based on the json schema, but 
that requires to add extra information in the JSON structure (to model the hierarchies and levels). In a similar vein, 
it would have been nice to abstract away the data generation to make it plug and pray with no customizations, but I don't
know if the javascript API exposes the necessary PDI hooks to do everything, and I'm too lazy to write this in Java.
