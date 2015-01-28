---
layout: post
title: "Test Kitchen Integration Helpers"
date: 2015-01-28 11:26:39 -0700
comments: false
categories: 
- Test Kitchen
---

## The Scenario
I was developing some serverspec tests against a Test Kitchen fixture cookbook designed to exercise multiple configurations of my LWRP. The tests loaded a spec_helper for general configuration as:
```ruby
require_relative '../../../kitchen/data/spec_helper'
```

This was an existing project so I continued the same pattern for my tests. The project test structure looked like this:
```
test/integration/my_suite/serverspec/assert_something_spec.rb # the test

test/shared/spec_helper.rb # the helper
```

As I was working back and forth in the tests and spec_helper I would execute `kitchen verify` to run the updated tests. I noticed that spec_helper.rb was not being updated, so I would have to:  
a) run `kitchen converge` or  
b) copy the file from my workstation to the kitchen node to the proper directory  
Both are inefficient methods.

## The Solution
Turns out the `kitchen verify` stage does update the spec_helper.rb as long as its in the correct directory. In the example of the serverspec tests I created the directory path `test/integration/helpers/serverspec` and moved spec_helper.rb file in the directory. Then I updated the tests to load spec_helper by adding:
```ruby
require_relative "./spec_helper"
```

The directory path convention will now do two things:  
1. `kitchen verify` will update the spec_helper.rb (and other helper files)  
2. spec_helper.rb will be copied into each serverspec suite directory allowing current
directory access.

## In Short
Place all Kitchen helper files for your test frameworks into `test/integration/helpers/<framework>` and your tests will have CWD access.
