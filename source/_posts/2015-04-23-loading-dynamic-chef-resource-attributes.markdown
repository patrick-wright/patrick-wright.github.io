---
layout: post
title: "Loading Dynamic Chef Resource Attributes"
date: 2015-04-23 09:56:38 -0700
comments: true
categories:
- LWRP
- Lazy Attribute Evaluation
- Recipe DSL
---

## The Scenario
I had created an LWRP that defined a standard attribute. That same attribute value
could also be assigned based on other parameters, and then assigned by the provider. For example:

#### Chef Resource snippet
```ruby
attribute :file_cache_path, kind_of: String, default: Chef::Config[:file_cache_path]

attribute :local_path, kind_of: String
```

#### Chef Provider snippet
```ruby
artifact = some_method_that_queries_for_an_artifact

new_resource.local_path(::File.join(new_resource.file_cache_path,
  ::File.basename(artifact.download_uri))) if new_resource.local_path.nil?
```

This statement will set `local_path` on `new_resource` if it was not set by the resource, and it will be assigned
a valid path dynamically generated during the chef client run.

#### Chef Recipe Option 1 - Common Approach
```ruby
path = '/tmp/file.path'

# assume this resource accepts a project name,
# queries for the latest version,
# and downloads the file to `local_path`
artifactory_artifact 'my-artifact'
  local_path path
end

package 'my-artifact' do
  source path
end
```

#### Chef Recipe Option 2 - Initial Expectation
```ruby
artifact = artifactory_artifact 'my-artifact'

package 'my-artifact' do
  source artifact.local_path
end
```

This example fails.  `artifact` will be assigned at compile time.  Since `local_path` has not been
set, it will be set to the attribute's default value. In this case `nil`.  I had initially expected the provider code
snippet above to properly set `local_path` before realizing that it is being executed after `artifact` instance is
assigned.

## The Solution
I was able to pick the brain of my colleague, Joshua Timberman.  He enlightened me of the `resources` DSL recipe method.
Read more about the [resources](http://docs.chef.io/dsl_recipe.html#resources) method.
```ruby
my_resource 'name' do
  attr1 :value
end


resources('my_resource[name]').attr1 => :value
resources('my_resource[name]').attr2 => nil # assume the provider sets this attribute
```

This will load the resource at compile time, but what we really want to do is load the resource during the chef client
run so we can load any changes that have been made by the provider.  We achieve this by calling the `lazy` attribute
evaluator. Read more about [lazy attribute evaluation](https://docs.chef.io/resources.html#lazy-attribute-evaluation).

```ruby
my_resource 'name'

lazy { resources('my_resource[name]').attr2 } => :dynamic_attribute
```

#### Chef Recipe Option 2 - Solution
```ruby
artifactory_artifact 'my-artifact'

package 'my-artifact' do
  source lazy { resources('artifactory_artifact[my-artifact]').local_path }
end
```

The `source` attribute will now be set to the `local_path` value dynamically assigned to the `new_resource`.

#### Chef Recipe Option 2 - Optimized
Recalling the syntax `lazy { resources('artifactory_artifact[my-artifact]').local_path }` could be quite annoying.
Add a helper library method. For this example `libraries/helpers.rb` will do.

```ruby
def artifactory_artifact_local_path(name)
  lazy { resources("artifactory_artifact[#{name}]").local_path }
end
```

Now call that method in the recipe.
```ruby
artifactory_artifact 'my-artifact'

package 'my-artifact' do
  source artifactory_artifact_local_path('my-artifact')
end
```

Makes me meta-wonder...
```ruby
some_resource 'resource-name' do
  attribute_name Chef::LazyResourceAttr.some_resource('resource-name', :dynamic_attribute)
end
```

Another project for another day.
