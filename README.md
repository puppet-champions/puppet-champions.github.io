This is the source repository for https://puppet-champions.github.io.

It's a GitHub Pages / Jekyll site, which means that merged changes are
published automatically. Profiles are managed semi-automatically.

When you add new users, first ensure that they're in the proper teams. All
members should belong to the Champions team, and League members should also
belong to that team. Once you're done with membership changes, run `rake sync`
at the command line and the necessary profiles will be generated.

Each profile starts from _template.erb and is populated with user information
about the new member. It's then dropped into a pull request and the user
is assigned to review, edit, and approve it. Once you, as an admin, merge
that PR, the site will be rebuilt to include the profile.

User profiles are synced from Nimble weekly using GitHub Actions.

![](https://github.com/puppet-champions/puppet-champions.github.io/workflows/Sync%20Champions/badge.svg)



## Requirements:

* You must have the `octokit` gem installed
* You must have this repository cloned locally
* You must have a [GitHub API token](https://help.github.com/en/articles/creating-a-personal-access-token-for-the-command-line).

