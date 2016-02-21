# Chef, The Berkshelf Way

[The Berkshelf Way – Jamie Winsor](https://www.chef.io/blog/chefconf-talks/the-berkshelf-way-jamie-winsor/) (2013).

## Background

The old way is to think of nodes as the first-class citizen. You always start with a node. Then you run chef-client to get the configuration for that node. Chef Server decides which cookbooks fit the version constraints and delivers them in a batch. The cookbooks are merely interlocking components of The Chef Run.

In contrast, The Berkshelf Way elevates cookbooks to standalone, self-contained applications that can be executed on their own schedule.

When you start treating cookbooks as applications, you will see that many of the principles from [The Twelve-Factor App](http://12factor.net/) methodology are applicable.

## Differences

- Version Indepdendence

    Imagine you have two cookbooks ("A" and "B") that need to run on the same node, and they both depend on a third cookbook "C". The maintainers of "C" release a new major update which is not backward compatible. But if you want to use it, then both "A" and "B" need to be updated simultaneously.

    But when your cookbook becomes a self-contained application, you can update "A" and "B" each on their own schedule without worrying about conflicting version constraints.

- Scheduling Diversity

    If your only way to use Chef is by kicking off a massive monolithic Chef run, then running Chef twice an hour sounds reasonable enough.

    But what about those times when just want something done fast? Maybe you're deploying an application, and you want to ensure that the configuration file is up to date. Do you want to spend 120 seconds for chef-client to run on each node, or do you want spend 10 seconds to run only the portions that apply to your application?

    And some things just don't need to happen that often. Maybe you have a Chef cookbook that installs some favourite tools for interactive SSH users. That would probably be fine to run on a nightly basis.

- Docker Alignment

    Want to start using containers? Docker is an application-first technology, so The Berkshelf Way is better aligned with it. If you need to provision dependencies and configuration files inside a container before deploying it, using Chef with a lightweight application cookbook will do just fine.

## Do I really need separate repos?

Some people find it easier to embrace The Berkshelf Way when they create a new git repository for each cookbook, but that is not strictly required. There are clever ways to use [Berkshelf with multiple cookbooks](http://chef.opscode.narkive.com/z3bZlv9b/single-repo-vs-repo-per-cookbook) in a single git repo. If you have already embraced The Berkshelf Way, there are only a few benefits to be gained by moving to a single repo per cookbook.

- You can bring the configuration code much closer to application code through submodules or even by putting the cookbook into the application repo.
- You avoid tag confusion, because git tags are global.
- Merges are guaranteed to contain only relevant commits.
- You can use simplified build scripts.

## Style

- Every cookbook has its own repository.

- Create a git tag every time you upload to Chef Server or Supermarket.

- Create a `release` branch to use for automated releases. When you're ready to release changes, merge them into this branch.

    Trigger a new release to Chef Server and/or Supermarket every time this branch is updated.

    Your merged changes must also have incremented the version number. Your build system should fail the build if the version has not been bumped.

    If you're uploading cookbooks to Chef Server, use `--freeze` as extra insurance to guarantee that you never overwrite a previous cookbook artifact.

# Unresolved

- If you have an application and you're creating a cookbook for it, does it make sense to keep the application and its cookbook in the same repository? Seems like it would be nice to be able to update the application code and dev configuration files and chef cookbook all in one go. But then you end up with a polluted tag history, changesets that include irrelevant changes, etc. Could git submodules make a difference here?

