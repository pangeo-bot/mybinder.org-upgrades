# mybinder.org-upgrades

A simple Python bot to keep [mybinder.org](https://mybinder.org) up-to-date with the latest versions of [repo2docker](https://github.com/jupyter/repo2docker) and [binderhub](https://github.com/jupyterhub/binderhub). The bot simply follows the process outlined in the [SRE documentation](https://mybinder-sre.readthedocs.io/en/latest/), specifically "[How to deploy a change](https://mybinder-sre.readthedocs.io/en/latest/deployment/how.html)".

At a high level:

- Diff the current commit hash for both repo2docker and binderhub repos with the deployed versions in the mybinder.org repo
- If different, fork the respective repo
- Clone the fork locally
- Checkout a new branch for the bump
- Make the appropriate edit to update the commit hash in the mybinder.org repo
- Add and commit the change
- Push to the branch in the forked repo
- Create a PR to the main mybinder.org repo
- Remove the locally cloned repo

Additionally, the bot checks if it can update an existing PR instead of creating new ones for the version bumps.

If no PRs exist (or they have been merged), the fork is removed.