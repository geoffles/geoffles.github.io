---
layout:	post
title:	"Attribute Precedence in Test Kitchen"
categories:	Chef
date:	2017-04-26 09:28:00+0200
tags:	[chef, kitchen]
---

***Abstract:** The order of precedence for test kitchen test suite attributes*

# Introduction

Whilst it's great that we can define attribute values for our cookbook attributes, Chef has a somewhat elaborate scheme when it comes to defining the values for those attributes. Chef defines [15 different orders of precedence](https://docs.chef.io/attributes.html#attribute-precedence), but not one of them is test-kitchen.

**So, in what order of precedence are kitchen attributes actually applied?**

Turns out test kitchen applies attributes at the end of the `force_default` level.

# The Test

The test case is pretty simple: Define a cookbook, role and environment which has an attribute at each precedence and define a test suite which sets each attribute as well.

## Overview

The structure is pretty normal:

~~~
|-- cookbooks
|   |-- testcookbook
|   |   |-- attributes
|   |   |   |-- default.rb
|   |   |-- recipes
|   |   |   |-- default.rb
|-- roles
|   |-- testrole.json
|-- environments
|   |-- testenvironment.json
|-- kitchen.yml
~~~

## Setting the attributes

Your cookbook attributes are pretty straightforward: Set one at each level in the respective recipe and attributes files. Eg:

`attributes/default.rb:`

{% highlight ruby %}
node.default['default_attribute'] = 'attribute1'
node.force_default['force_default_attribute'] = 'attribute5'
{% endhighlight %}

`recipes/default.rb:`

{% highlight ruby %}
node.default['default_recipe'] = 'attribute2'
node.force_default['force_default_recipe'] = 'attribute6'
{% endhighlight %}

For the roles and environments though, you'll need to specify these in your kitchen file. The role is as simple as setting the role in the runlist, but the environment requires a little more knowledge of test kitchen.

You need to set the environment in the provsioner section:

{% highlight ruby %}
provisioner:
  name: chef_zero
  require_chef_omnibus: 12.19.36
  client_rb: 
    environment: testEnvironment
{% endhighlight %}

And of course you must create a suite in the test kitchen and set the attributes:

{% highlight ruby %}
suites:
  - name: testAttributes
    run_list:
      - role[testRoles]
      - recipe[testAttributeCookbook]
    attributes:
      test_dependency:
        environment: testEnvironment
        attribute1: testKitchen
        attribute2: testKitchen
        attribute3: testKitchen
        attribute4: testKitchen
        attribute5: testKitchen
        attribute6: testKitchen
        attribute7: testKitchen
        attribute8: testKitchen
        attribute9: testKitchen
        attribute10: testKitchen
        attribute11: testKitchen
        attribute12: testKitchen
        attribute13: testKitchen
        attribute14: testKitchen
{% endhighlight %}

And finally, you need to of course output the results in your recipe:

{% highlight ruby %}
puts node['test_dependency']['attribute1']
puts node['test_dependency']['attribute2']
puts node['test_dependency']['attribute3']
puts node['test_dependency']['attribute4']
puts node['test_dependency']['attribute5']
puts node['test_dependency']['attribute6']
puts node['test_dependency']['attribute7']
puts node['test_dependency']['attribute8']
puts node['test_dependency']['attribute9']
puts node['test_dependency']['attribute10']
puts node['test_dependency']['attribute11']
puts node['test_dependency']['attribute12']
puts node['test_dependency']['attribute13']
puts node['test_dependency']['attribute14']
{% endhighlight %}

# Results

You're now ready to run a `kitchen converge`, and should note the following in the output:

~~~
testKitchen
testKitchen
testKitchen
testKitchen
testKitchen
testKitchen
attribute7
attribute8
attribute9
attribute10
attribute11
attribute12
attribute13
attribute14
~~~

Reconciling the attribute numbers back, we note that `attribute7` is `normal` level attributes, indicating that test kitchen applies itself as `force_default` level after the cookbooks and recipes (or `normal` before the cookbook). 

# Conclusion

We've run a test to determine at what level the attributes in test kitchen are applied at and found them to happen between `force_default` and `normal`.
