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

Assigning a Chef resource to an instance variable can be quite useful when you need to recall attribute values for later use. However, only the default and set attributes will be available to the resource instance variable. If the provider sets the value of an attribute during the client run the new value for that attribute will not be available to the resource instance variable. In this post, I cover how to load attributes that are set dynamically. Below are simple examples to demonstrate the problem.

#### Chef Recipe Option 1 - Static Approach
This example demonstrates a straight-forward use case where the path of a file has been predetermined.
```ruby
path = '/tmp/file'

remote_file path
  source "some_remote_path"
end

package 'downloaded-package' do
  source path
end
```

#### Chef Recipe Option 2 - Initial Expectation
To describe a more dynamic example I have provided some code. Here is a Resource class with code removed for brevity containing an attribute that can be set on the resource, or in relation to our issue can be set by the provider.
```ruby
class Chef::Resource::OmnibusArtifactoryArtifact < Chef::Resource::LWRPBase
  self.resource_name = :omnibus_artifactory_artifact

  actions :create, :create_if_missing
  default_action :create

  # ...

  #
  # download directory
  #
  attribute :file_cache_path, kind_of: String, default: Chef::Config[:file_cache_path]

  #
  # local file path. Can set the canonical file path or call #local_path on a
  # resource instance to retrieve the fetched file name
  #
  attribute :local_path, kind_of: String

  # ...
end
```

Here is the relative Provider class with code removed for brevity. Notice the statement in the the `#download` method where `new_resource.local_path` is assigned. If it was not set by the resource it will be assigned a valid path dynamically generated during the chef client run.
```ruby
aclass Chef::Provider::OmnibusArtifactoryArtifact < Chef::Provider::LWRPBase

  # ...

  action :create do
    download :create
  end

  action :create_if_missing do
    download :create_if_missing
  end

  private

  # ...

  def omnibus_artifact_property_search
    # ...
    # returns an Arifact instance based on set attributes
  end

  def download(remote_file_action)
    artifact = omnibus_artifact_property_search
    properties = artifact.properties

    # set local_path if not set as an attribute
    new_resource.local_path(::File.join(new_resource.file_cache_path,
                            ::File.basename(artifact.download_uri))) if new_resource.local_path.nil?

    converge_by("Download #{artifact.download_uri} to #{new_resource.local_path}") do
      remote_file new_resource.local_path do
        action remote_file_action
        source artifact.download_uri
        # ...
      end
    end
  end
 
  # ...

end
```

Recipe example:
```ruby
artifact = artifactory_artifact 'my-artifact'

package 'my-artifact' do
  source artifact.local_path
end
```

This example fails. `artifact` will be assigned at compile time.  Since `local_path` has not been
set, it will be set to the attribute's default value. In this case `nil`.

_To my mistake, I had initially expected the provider code snippet above to properly set `local_path` before realizing that it is being executed after `artifact` instance is assigned._

## The Solution
I was able to pick the brain of my colleague, Joshua Timberman.  He enlightened me of the `resources` DSL recipe method. This method will look up a resource in the resource collection using the sytnax `resources("resource_type[resource_name]")`. Since it returns a Resource instance we can call methods on that instance. For example, calling the methods that return attribute values.
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
