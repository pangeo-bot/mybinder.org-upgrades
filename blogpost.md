# Automating mybinder.org dependency upgrades

As both BinderHub and repo2docker continue rapid development as standalone packages, the service of mybinder.org continues its own growth in offering a browser-based exploration platform for Jupyter notebooks. Depending on BinderHub and repo2docker for hte latest functionality demands that mybinder.org be continuously upgraded. Moreover, to avoid merging in massive updates at irregular intervals, it is desirable to merge updates in frequent intervals of smaller changes in order to more easily identify any breaking changes from the depdencies.

For some time, the mybinder.org community relied on continuous updates from its developers following a process outlies in the "[Site Reliability Guide](https://mybinder-sre.readthedocs.io/en/latest/)". While there are several tasks necessary to simply keep the site running smoothly, one tedious and reptitive task is simply [updating commit SHAs](https://mybinder-sre.readthedocs.io/en/latest/deployment/how.html) for BinderHub and repo2docker whenever changes were made to the main repositories.

Recently, the team has built a bot to automate this process, and we've decided to highlight its functionality in this blogpost!

At a high level, this is what we want our bot to do:

- Diff the current commit hash for both repo2docker and BinderHub repos with the deployed versions in the mybinder.org repo. If either or both are different:
	- Fork the respective repo
	- Clone the fork locally
	- Checkout a new branch for the bump
	- Make the appropriate edit to update the commit hash in the mybinder.org repo
	- Add and commit the change
	- Push to the branch in the forked repo
	- Create a PR to the main mybinder.org repo
	- Remove the locally cloned repo

Additionally, the it would be ideal if the bot could update an existing PR instead of creating new ones for the version bumps. We'd also like to provide some information in the comments of the PR as to what high level changes were made so we have some idea about what we're merging in.

Now that we've broken it down a bit, let's write up some Python code that can do this. Once we have a functioning script, we can worry about how we will run this in the cloud (cronjob vs. webapp).

# Writing the bot.

## Step 1: Retrieve current mybinder.org versions

As we said above, the first step is to see if any changes are necessary in the first place. Fortunately, [@choldgraf](https://github.com/choldgraf) had already made a [script](https://github.com/jupyterhub/mybinder.org-deploy/blob/master/scripts/list_new_commits.py) to do this.

To find the current live commit SHA for BinderHub in mybinder.org, we simply check the [`requirements.yaml`](https://raw.githubusercontent.com/jupyterhub/mybinder.org-deploy/master/mybinder/requirements.yaml) file:

```python
from yaml import safe_load as load
import requests
url_requirements = "https://raw.githubusercontent.com/jupyterhub/mybinder.org-deploy/master/mybinder/requirements.yaml"
requirements = load(requests.get(url_requirements).text)
binderhub_dep = [ii for ii in requirements[
    'dependencies'] if ii['name'] == 'binderhub'][0]
bhub_live = binderhub_dep['version'].split('-')[-1]
print(bhub_live)
```

Similarly, for repo2docker, we check the mybinder [`values.yaml`](https://raw.githubusercontent.com/jupyterhub/mybinder.org-deploy/master/mybinder/values.yaml) file:

```python
url_helm_chart = "https://raw.githubusercontent.com/jupyterhub/mybinder.org-deploy/master/mybinder/values.yaml"
helm_chart = requests.get(url_helm_chart)
helm_chart = load(helm_chart.text)
r2d_live = helm_chart['binderhub']['config'][
    'BinderHub']['build_image'].split(':')[-1]
print(r2d_live)
```

Let's store these in a dictionary we can use for later reference:

```python
commit_info = {'repo2docker': {}
               'binderhub': {}}

commit_info['binderhub']['live'] = bhub_live
commit_info['repo2docker']['live'] = r2d_live
```

## Step 2: Retrieve lastest commits

When we get the latest commit SHAs for repo2docker and BinderhHub, we need to be careful and make sure we don't automatically grab the latest one from GitHub. The travis build for mybinder.org looks for the repo2docker Docker image from Docker Hub, and the latest BinderHub from the jupyterhub heml chart.

Let's get the repo2docker version first:

```python
url = "https://hub.docker.com/v2/repositories/jupyter/repo2docker/tags/"
resp = requests.get(url)
r2d_master = resp.json()['results'][0]['name']

# add to commit_info dictionary
commit_info['repo2docker']['latest'] = r2d_master
print('repo2docker', commit_info['repo2docker']['live'], commit_info['repo2docker']['latest'])
```

Now we can do BinderHub:

```python
url_helm_chart = 'https://raw.githubusercontent.com/jupyterhub/helm-chart/gh-pages/index.yaml'
helm_chart_yaml = load(requests.get(url_helm_chart).text)

# sort by date created
updates_sorted = sorted(helm_chart_yaml['entries']['binderhub'], key=lambda k: k['created'])

# add to commit_info dictionary
commit_info['binderhub']['latest'] = updates_sorted[-1]['version'].split('-')[-1]
print('binderhub', commit_info['binderhub']['live'], commit_info['binderhub']['latest'])
```

Great, now we should have all the information we need to determine *whether* and update needs to be made or now, *and* what the new commit SHA should be!

# Step 3: Fork repo

If we determine an upgrade for the repo is necessary, we need to fork the mybinder.org repository, make the change, commit, push, and make a PR. Let's just make a fork first. Fortunately, the GitHub API has all the functionality we need for all of this:

```python
TOKEN = 'YOUR GITHUB ACCOUNT SECRET TOKEN'
res = requests.post('https://api.github.com/repos/jupyterhub/mybinder.org-deploy/forks',
            headers={'Authorization': 'token {}'.format(TOKEN)})
```

Using the API for a post request to the `forks` endpoint will fork the repo to your account. That's it!


# Step 4: Clone your fork

You should be quite used to this! We'll use Python's `subprocess` module to run all of our `bash` commands:

```python
subprocess.check_call(['git', 'clone', 'https://github.com/henchbot/mybinder.org-deploy'])
```

Let's also `cd` into it and check out a new branch:

```python
os.chdir('mybinder.org-deploy')
subprocess.check_call(['git', 'checkout', '-b', 'repo2docker_bump'])
```

# Step 5: Make the file changes

Now we need to acutally edit the file like we would for an upgrade.

For repo2docker:

```python
with open('mybinder/values.yaml', 'r') as f:
    values_yaml = f.read()

updated_yaml = values_yaml.replace(
    "jupyter/repo2docker:{}".format(
        commit_info[upgrade]['live']),
    "jupyter/repo2docker:{}".format(
        commit_info[upgrade]['latest']))   

fname = 'mybinder/values.yaml'
with open(fname, 'w') as f:
    f.write(updated_yaml)
```

For BinderHub:

```python
with open('mybinder/requirements.yaml', 'r') as f:
    requirements_yaml = f.read()

updated_yaml = requirements_yaml.replace(
    "version: 0.2.0-{}".format(commit_info[upgrade]['live']),
    "version: 0.2.0-{}".format(commit_info[upgrade]['latest']))   

fname = 'mybinder/requirements.yaml'
with open(fname, 'w') as f:
    f.write(updated_yaml)
```

# Step 6: Stage, commit, push

Now that we've edited the correct files, we can stage and commit the changes:

```python
# use var fname from editing step
subprocess.check_call(['git', 'add', fname])

if repo == 'repo2docker':
    commit_message = 'repo2docker: https://github.com/jupyter/repo2docker/compare/{}...{}'.format(
        self.commit_info['repo2docker']['live'],commit_info['repo2docker']['latest'])
elif repo == 'binderhub':
    commit_message = 'binderhub: https://github.com/jupyterhub/binderhub/compare/{}...{}'.format(
        self.commit_info['binderhub']['live'], commit_info['binderhub']['latest'])

subprocess.check_call(['git', 'config', 'user.name', 'henchbot'])
subprocess.check_call(['git', 'config', 'user.email', 'henchbot.github@gmail.com'])
subprocess.check_call(['git', 'commit', '-m', commit_message])
subprocess.check_call(['git', 'push', 'https://henchbot:{}@github.com/henchbot/mybinder.org-deploy'.format(TOKEN), repo+'_bump'])
```

Awesome, we now have a fully updated fork ready to make a PR to the main repo!

# Step 7: Make the body for the PR

We want the PR to have a nice comment explaining what's happening and linking any helpful information so that the merger knows what they're doing.

```python
if repo == 'repo2docker':
    compare_url = 'https://github.com/jupyter/repo2docker/compare/{}...{}'.format(
                        commit_info['repo2docker']['live'], 
                        commit_info['repo2docker']['latest'])
    body = '\n'.join(['This is a repo2docker version bump. See the link below for a diff of new changes:\n', compare_url + ' \n'])

elif repo == 'binderhub':
    compare_url = 'https://github.com/jupyterhub/binderhub/compare/{}...{}'.format(
                        commit_info['binderhub']['live'], 
                        commit_info['binderhub']['latest'])
    body = '\n'.join(['This is a binderhub version bump. See the link below for a diff of new changes:\n', compare_url + ' \n'])
```

# Step 8: Make the PR

```python
pr = {
    'title': '{}: {}...{}'.format(repo,
                                  commit_info[repo]['live'],
                                  commit_info[repo]['latest']),
    'body': body,
    'base': 'master',
    'head': 'henchbot:{}_bump'.format(repo)}

res = requests.post('https://api.github.com/repos/jupyterhub/mybinder.org-deploy/pulls',
    headers={'Authorization': 'token {}'.format(TOKEN)}, json=pr)
```

# Step 9: Confirm and merge!

Check out the mybinder.org PRs, you should now see your automated PR! If it looks good, it can be merged.

# Step 10: Automating your script (cronjob)

Now that you have a script you can simply execute to create a PR, we want to make this as hands-off as possible. Generally you have two options: (1) set this script to be run on a cronjob; (2) have a webapp listener that gets pinged whenever a change is made and executes your script.

Given that these aren't urgent updates that need to be made seconds or minutes after a repository update, we will go for the easier option of a cronjob.

If you aren't familiar with cronjob, it's simply a system program that will run whatever command you want at whatever time or time interval you want. For now, we've decided that we want to execute this script every hour. Since I have a few projects going on, I like to keep the crontab jobs in a file:

```
vim crontab-jobs
```

You can define your cronjobs here with the correct sytnax (space-separated). Since we want to run this every hour, we will set it to run on the 0 minutes, for every hour, every day, every month, every year. Check out [this site](https://crontab.guru/every-1-hour) for help with the crontab syntax.

```
0 * * * * cd /home/pi/projects/mybinder.org-upgrades && HENCHBOT_TOKEN='XXXXX' /home/pi/miniconda3/bin/python henchbot.py
```

Now we point our crontab to the file we've created:

```
crontab crontab-jobs
```

To see our active crontab, we can list it:

```
crontab -l
```

That's it!
