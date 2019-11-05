This is the source repository for https://puppet-champions.github.io.

It's a GitHub Pages / Jekyll site, which means that merged changes are
published automatically. Profiles are managed semi-automatically.

Each profile starts from `_template.erb` and is populated with user information
about the new member. It's then dropped into a pull request and the user
is assigned to review, edit, and approve it. This also happens when you
promote or demote a Champion between tiers, or when you remove a Champion.
Once an admin merges that PR, the site will be rebuilt to reflect changes.

_**Note:** The site will not be updated until you merge Pull Requests!_

Users are managed in Nimble. On the _Community_ tab, set their appropriate
_Puppet Champion_ status and make sure that they've got a GitHub username
set. User profiles are synced weekly using GitHub Actions.

This badge demonstrates the status of the sync process. If it's red, then
we need to pay attention. Take a look at the latest sync task on the _Actions_
tab above to see what the warnings are. Generally it means that a Champion
doesn't have a GitHub username associated, or that a Champion has not yet
approved their profile page.

![](https://github.com/puppet-champions/puppet-champions.github.io/workflows/Sync%20Champions/badge.svg)



## Notes:

* If authentication is failing for Nimble, then ensure that the team has a current API token.
See the [Nimble developer docs page](http://support.nimble.com/en/articles/822159-generate-an-api-key-to-access-the-nimble-api)
for more information.
* If a profile is inadvertently changed or removed, you can undo that change by finding
  the associated PR and using the [`revert` functionality](https://help.github.com/en/github/collaborating-with-issues-and-pull-requests/reverting-a-pull-request).
