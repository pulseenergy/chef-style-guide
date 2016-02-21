# Chef Style Guide

*And other assorted best practices when using Chef.*

## Table of Contents
1. [Other Style Guides](#other-style-guides)
1. [Repository Structure](#repository-structure)
1. [Attributes](#attributes)
1. [Cookbooks](#cookbooks)
1. [Providers](#providers)
1. [Recipes](#recipes)
1. [Templates](#templates)
1. [Conditionals](#conditionals)
1. [Environments](#environments)
1. [Roles](#roles)
1. [Data Bags](#data-bags)
1. [Nodes](#nodes)
1. [Test Kitchen](#test-kitchen)
1. [Secrets](#secrets)
1. [Debugging](#debugging)
1. [Common](#common)

## Other Style Guides

- Read [Patterns To Follow](https://docs.chef.io/ruby.html#patterns-to-follow) in the official Chef Style Guide. Some points have been copied into this guide.

- Foodcritic is an excellent lint tool for Chef cookbooks. This style guide will skip over anything that is already automated by Foodcritic.

## Repository Structure

- Create a new git repository for each cookbook. This is known as [The Berkshelf Way](the_berkshelf_way/).

    Not so sure about this? Or you're stuck maintaining Chef with many cookbooks in one repository? We still have some guidance on how to make the best of using a [monorepo](monorepo/).

## Attributes

- Never use `node.set` or `node.normal`. Creating persistent node attributes is almost universally a bad idea.

    A normal Chef run starts with an empty set of attributes and populates them based on cookbooks, environments, and roles, all of which are visible in the source code.

    Attributes persisted onto a node are never visible in source code. They can produce confusing behaviour that is difficult to troubleshoot. The stored node attributes will impact all future runs until you manually remove them from the node.

    > "Normal and override attributes are cleared at the start of the chef-client run, and are then rebuilt as part of the run based on the code in the cookbooks and recipes at that time." - [Patterns To Follow](https://docs.chef.io/ruby.html#patterns-to-follow)

- All attributes defined by your cookbook should exist within your cookbook's namespace.

    ```ruby
    node['my_cookbook']['my_attribute'] = 'foo'
    ```

- If you need to read attributes from another cookbook, copy them into the namespace of your own cookbook. Do this inside `attributes/external.rb`. Try to minimize the number of external attribute accesses.

    Use `include_attribute` to create sections within your `external.rb`. Although `include_attribute` is only a hint to the Chef compiler, it makes it much easier to search for dependencies in a code base when you want to refactor a cookbook.

    ```ruby
    include_attribute 'other_cookbook::default'
    node['my_cookbook']['limit'] = node['other_cookbook']['limit']
    ```

- If you are writing a wrapper cookbook, then it's okay to put your overrides into `default.rb`. If you need to access any other cookbooks other than the one being wrapped, those should go into `attributes/external.rb`.

- Avoid derived attributes in attribute files.

    ```ruby
    # Attributes:: default
    // DO NOT DO THIS
    node['my_cookbook']['version'] = '1.4.8'
    node['my_cookbook']['url'] = "http://mysite/#{node['my_cookbook']['version']}.tar.gz"
    ```

    If somebody overrides the `version` attribute, the `url` attribute won't be recomputed. It doesn't matter whether the attribute is overridden in an environment file or by another cookbook.

    When you need to derive attributes, do it inside a recipe or provider.

## Cookbooks

- Use [Semantic Versioning](http://semver.org/) on any cookbook that is uploaded to Chef Server or published to Supermarket. Every cookbook declares a public API, and thus it is appropriate to use SemVer.

    If you aren't using Test Kitchen, this means bumping the patch version every time you try a new change. You should be using Test Kitchen.

- Never *ever* decrement the version of a cookbook. Failure to adhere is a violation of [Rule #2 in SemVer 2.0](http://semver.org/#spec-item-2).

    Chef-client will always use the highest-numbered cookbook that is available after considering all constraints. If Chef Server knows about a cookbook with a higher number than the one you just uploaded, then your code is not going to get run. Do not add a version constraint in your test environment to work around this; it will definitely bite you later on.

    Your build system should fail the build if the cookbook version has not been incremented beyond the last uploaded cookbook. This matters even more if you're publishing to Supermarket.

- Always freeze your cookbooks when uploading them. Failure to do so is a violation of [Rule #3 in SemVer 2.0](http://semver.org/#spec-item-3).

    ```
    knife cookbook upload my_cookbook --freeze
    ```

- Include a `CHANGELOG.md`. Focus on clear documentation of breaking changes. Be terse with everything else. Any changes that are not backward-compatible should bump the major version and warrant an entry in the CHANGELOG so that people know how to become compatible with the latest version.

- If you have two cookbooks with many dependencies that need to obtain a shared attribute, consider moving that attribute into a new lightweight cookbook. This helps to avoid gigantic dependency chains that Berkshelf will struggle to resolve.

## Providers

- Always `use_inline_resources` at the start of your provider. This is a kind of "strict mode". This is expected to become the default behaviour in Chef 13.

    > "use_inline_resources is used to separate a LWRP's run context from the main run, making it run in an isolated mini-run. You cannot subscribe to / notify resources that are not part of this resource's context." - [Backslasher](http://blog.backslasher.net/chef-inline.html)

- Use providers instead of recipes when writing reusable cookbooks. Providers have the benefit that they can be instantiated multiple times within a single Chef run. This comes up more often than you would expect.

- Avoid unnecessary recipes in your provider cookbooks. They only serve as a distraction and maintenance burden.

## Recipes

- Leave the `default.rb` recipe empty. Avoid the tempation to include sub-recipes inside `default.rb`. Create other recipes with specific functions. Recipes are the main public interface of your cookbook, and consumers are expected to know about them.

    > "Don’t use the default recipe (leave it blank). Instead, create recipes called server or client (or other)." - [Patterns To Follow](https://docs.chef.io/ruby.html#patterns-to-follow)

- Create a `common.rb` recipe if there are multiple sub-recipes that would otherwise define the same resources (eg. a directory). Otherwise you might find yourself running into [CHEF-3694](http://tickets.chef.io/browse/CHEF-3694) warnings.

## Templates

- Avoid using the `node` object within templates.

    If you need to use attributes in a template, add them to the template resource using the `variables` parameter.

    ```erb
    // GOOD:
    Alice is <%= @age %> years old.
    // BAD:
    Alice is <%= node['my_cookbook']['alice']['age'] %> years old.
    ```

    Never refer to attributes from other cookbooks in your template.

## Conditionals

- If you need to do something environment-specific, use an attribute instead. Never make conditionals dependent on the name of an environment.

- Understand [Chef's Two Pass Model](https://coderanger.net/two-pass/). Be aware of the difference between `if` statements and guards. One of them is handled during the compile phase, and the other during the execute phase.

## Environments

- Environment files are great. Prefer them instead of data bags. Everything in the environment is available at the beginning of the run, whereas data bags require additional HTTP requests.

- Avoid keeping secrets in Chef. If you must keep them in Chef, then environment files are the natural place for them. But consider using something like [Vault](https://github.com/hashicorp/vault) instead.

- Avoid tracking the current addresses of backing services in your environment file. If you must keep them in Chef, environment files are the natural place. But consider using a proper Service Discovery tool like [Consul](https://www.consul.io/) or Zookeeper instead.

- Environment names are always uppercase.

    The name specified within the environment file should be an exact case-sensitive match of the filename.

    When you specify an environment with chef-zero it looks for a file with exact matching case. If the filename has different case than the usage in your recipes, you will have difficulty with matching and lookups. *(Although admittedly, you should never depend on the name of an environment within your code. Use attributes for that instead.)

## Roles

- Avoid setting attributes in roles. It's often confusing.

## Data Bags

- Do not use data bags to store data that is different across environments. Environment files are very good at this already. Having two places to look ends up being very confusing.

- Before making structural changes to a data bag you must make your cookbook forward-compatible, and the cookbook must be promoted to all environments.

    Do not try to modify a data bag and a cookbook simultaneously. Data bags are atomic, and so are cookbooks. If you try to modify them simultaneously, you will end up with converge failures or worse.

    1. Make your cookbook forward-compatible
    2. Promote your cookbook to all environments
    3. Change the data bag
    4. Simplify your cookbook
    5. Promote your cookbook to all environments

- Minimize the number of data bag lookups that occur. A new HTTP request is sent every time you request a data bag.

- Create a `data_bag.rb` recipe in each cookbook that needs to look up something from a data bag. Put all of your data bag lookups into this recipe. This helps to reduce duplicate lookups.

- Try to name your data bag so that it matches the name of your cookbook. But due to the shared nature of data bags, this is often not possible.

## Nodes

- Use only one entry in the run_list for each node. Node run lists do not have the benefit of version control, and become very difficult to maintain once they grow long enough.

- Use a `base` cookbook for the essentials that are needed on every single node.

    - If you have something that isn't needed on every node, it does not belong in base. A good example of what goes in a base cookbook is users and SSH configuration.
    - Only add your `base` cookbook to Chef roles, never to a node run_ _list. Nodes should have only one item in the run list, and it should be a role.
    - Application cookbooks should never depend on the base cookbook.
    - Always add your `base` cookbook to the **end** of your run_list. Adding it at the end instead of the beginning helps to ensure that application cookbooks don't have hidden dependencies on your `base` cookbook.
    - Try hard to minimize the number of cookbook dependencies used by your base cookbook.

## Test Kitchen

- Use Test Kitchen. Include a `.kitchen.yml` file with every cookbook. Make sure your cookbook converges locally before promoting or publishing your cookbook. This helps with http://12factor.net/dependencies.

- Always test against the latest version of Chef. You might want to pin a specific version of Chef to make Kitchen run faster, but remember to keep it updated.

- In `.kitchen.yml` all references to environments and data bags should be contained within the cookbook. This only applies if your cookbooks are stored in a monorepo. Any references outside the cookbook (ie. to a directory of shared environments and data bags) makes it too easy to overlook implicit dependencies. Using local environment and data bag files is an easy way to explicitly show and test those attribute dependencies.

## Secrets

- Try hard to avoid keeping secrets in Chef. Consider something like Hashicorp Vault.

## Debugging

- The very best output comes from executing chef-client with the debug log level while running as root user.

    ```
    chef-client --log_level debug
    ```

## Supermarket

- TODO: 

## Common

- Use ChefDK to install your development environment.

- All files should end with a linefeed. Vim does this for you. Sublime Text can be configured to do this, but does not do it by default.

- Define arrays on multiple lines, and put a comma at the end of every line. This is one of the best features of Ruby.

    ```ruby
    # roles/my_role.rb
    run_list [
      "recipe[my_app::deploy]",
      "recipe[my_base::all]",
    ]
    ```

- Avoid hardcoding node names (or patterns to match node names) anywhere in your recipes. Use attributes instead.

## Unresolved questions

- Hyphens or underscores?
- Where is the best place to apply version constraints?
- What should you do with Chef logfiles? Can they be shipped to Splunk or Logstash easily?
- Does anybody have great tips on how to avoid making a README.md that ends up looking terrible on Supermarket?

