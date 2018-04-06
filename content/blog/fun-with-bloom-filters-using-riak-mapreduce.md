+++
title = "fun with bloom filters using riak mapreduce"
date = "2011-09-14"
slug = "2011/09/14/fun-with-bloom-filters-using-riak-mapreduce"
Categories = []
+++

So I.ve been toying with some ideas on how to do large mapreduce jobs, and pushing the processing into Riak (Erlang) to make use of distributed processing and data-locality.

It took a while to figure out how to get this to work, but here it is.

First, attach to the riak console

``` bash Attaching to erlang
# riak attach
```

<!--more-->
Wait for the prompt, then you can test with the following code

``` erlang Playing with bloom filters inside Riak
{ok, Client} = riak:client_connect('riak@127.0.0.1').
B1 = bloom:add_element(<<"abcdef">>, bloom:bloom(100000)).
B2 = bloom:add_element(<<"boogey woogie">>, bloom:bloom(100000)).
Client:delete(<<"testbucket">>,<<"testkey2">>).
Client:delete(<<"testbucket">>,<<"testkey1">>).
Client:put(riak_object:new(<<"testbucket">>, <<"testkey">>, term_to_binary(B1))).
Client:mapred_bucket(<<"testbucket">>, 
    [{map, {qfun, fun(Object, undefined, Arg) -> 
        [
            {
                riak_object:key(Object), 
                bloom:is_element(Arg, binary_to_term(riak_object:get_value(Object)))
            }
        ] end}, <<"abcdefg">>, true}]).
Client:mapred_bucket(<<"testbucket">>, 
    [{map, {qfun, fun(Object, undefined, Arg) -> 
        [
            {
                riak_object:key(Object), 
                bloom:is_element(Arg, binary_to_term(riak_object:get_value(Object)))
            }
        ] end}, <<"abcdef">>, true}]).
Client:put(riak_object:new(<<"testbucket">>, <<"testkey2">>, <<"boogey woogie">>)).
Client:mapred_bucket(<<"testbucket">>, [{map, {qfun, fun(Object, undefined, Arg) -> 
    [
        {
            riak_object:key(Object), 
            bloom:is_element(riak_object:get_value(Object), binary_to_term(Arg))
        }
    ] end}, term_to_binary(B2), true}]).
```

This code tests a couple of things. First, we're testing evaluating a bloom filter that's stored inside a Riak object against a static argument that's passed on to the mapreduce job.


Next, we create a map reduce job that passes a bloom filter as an argument, and test the value of the Riak object against the bloom filter.


This code makes use of the bloom filter which is currently a module that.s exported inside riak_core. While with the 0.14 series of Riak, whole bucket operations are very inefficient, this is slated to be vastly improved with the 1.0 release (at least that.s what I.ve been told).


Should this pan out to be true, it should be possible to make very efficient queries like those above, and have all the processing done inside of Riak itself, which is pretty amazing.
