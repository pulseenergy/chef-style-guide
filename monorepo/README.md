# Chef with a Monorepo

- Any commit affecting a cookbook must only include changes to files within that cookbook.

    Do not fall into the trap of thinking that all commits to the monorepo are atomic, because they aren't. All those cookbooks and data bags are being deployed independently of each other.

    When you allow your commits to cross those boundaries, it's easy to forget that you need to be careful about maintaining compatibility between each piece of the system.

- Avoid downloading community cookbooks into your monorepo. Use Berkshelf or Chef Librarian to resolve and upload the dependencies.

    If you really need to download community cookbooks into your repository, then at least follow this one rule: Never *ever* make changes to community cookbooks in your repository. This will make it incredibly difficult to take advantage of upstream fixes and improvements.

- Read `metadata.json` in your scripts. You're going to have scripts, and they're probably going to care about your cookbook versions. Those cookbooks downloaded from Supermarket will not have a metadata.rb file.
