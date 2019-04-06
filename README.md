# Introduction
It's a fork of [truffleHog](https://github.com/dxa4481/truffleHog.git), introducing more flexibility to checks. Searches through git repositories for secrets, digging deep into commit history and branches. This is effective at finding secrets accidentally committed.

## Install
```
pip install forestHog
```

## foresthog
truffleHog previously functioned by running entropy checks on git diffs. This functionality still exists, but high signal regex checks have been added, and the ability to surpress entropy checking has also been added.

These features help cut down on noise, and makes the tool easier to shove into a devops pipeline.


```
forestHog --regex https://github.com/dxa4481/truffleHog.git
```

You can also check a repo directly from your file system:

```
forestHog file:///user/dxa4481/codeprojects/truffleHog/
```

To enable entropy check, use following:
```
forestHog --regex --entropy https://github.com/dxa4481/truffleHog.git
```

![Example](https://i.imgur.com/YAXndLD.png)

### Customizing

Custom regexes can be added with the following flag `--rules /path/to/rules`. You can also add regexes along with default ones using `--add-rules /path/to/rules` flag. It makes it easier to extend the rule checks while using default and custom rules both. File provided by `--rules` or `--add-rules` should be a json file of the following format:
```
{
    "RSA private key": "-----BEGIN EC PRIVATE KEY-----"
}
```
Things like subdomain enumeration, s3 bucket detection, and other useful regexes highly custom to the situation can be added.

Feel free to also contribute high signal regexes upstream that you think will benefit the community. Things like Azure keys, Twilio keys, Google Compute keys, are welcome, provided a high signal regex can be constructed.

trufflehog's base rule set sources from https://github.com/dxa4481/truffleHogRegexes/blob/master/truffleHogRegexes/regexes.json  

You can also check what regexes will the program check against before actually running it against your repo. This is a helpful check to make sure your custom rules/regexes are detected successfully:
```
forestHog --regex --show-regex <git_url>
```
A json response will be returned. A sample is shown below:
```
{
  "Slack Token": "(xox[p|b|o|a]-[0-9]{12}-[0-9]{12}-[0-9]{12}-[a-z0-9]{32})",
  "RSA private key": "-----BEGIN RSA PRIVATE KEY-----",
  ...
}
```
  
Entropy is checked at a minimum of 20-letter words. You can control the word-length and threshold value for the entropy checks to your liking.  
`--entropy-wc` controls the word-length. [default: 20]  
`--entropy-hex-thresh` controls the threshold for entropy calculated for hex strings. [default: 3.0]  
`--entropy-b64-thresh` controls the threshold for entropy calculated for base64 strings. [default: 4.5]  

### How it works
This module will go through the entire commit history of each branch, and check each diff from each commit, and check for secrets. This is both by regex and by entropy. For entropy checks, forestHog will evaluate the shannon entropy for both the base64 char set and hexidecimal char set for every blob of text greater than `--entropy-wc` characters comprised of those character sets in each diff. If at any point an entropy crosses the thresholds defined by `--entropy-hex-thresh` and `--entropy-b64-thresh` for a string greater than `--entropy-wc` characters, it will print to the screen.

### Help Dialog

```
usage: forestHog [-h] [--json] [--show-regex] [--regex] [--rules RULES]
                     [--add-rules ADD_RULES] [--entropy]
                     [--entropy-wc ENTROPY_WC]
                     [--entropy-b64-thresh ENTROPY_B64_THRESH]
                     [--entropy-hex-thresh ENTROPY_HEX_THRESH]
                     [--since-commit SINCE_COMMIT] [--max-depth MAX_DEPTH]
                     [--branch BRANCH] [--repo-path REPO_PATH] [--cleanup]
                     git_url

Find secrets hidden in the depths of git.

positional arguments:
  git_url               URL for secret searching

optional arguments:
  -h, --help            show this help message and exit
  --json                Output in JSON
  --show-regex          prints out regexes that will computed against repo
  --regex               Enable high signal regex checks
  --rules RULES         Ignore default regexes and source from json list file
  --add-rules ADD_RULES
                        Adds more regex rules along with default ones from a
                        json list file
  --entropy             Enable entropy checks
  --entropy-wc ENTROPY_WC
                        Segments n-length words to check entropy against
                        [default: 20]
  --entropy-b64-thresh ENTROPY_B64_THRESH
                        User defined entropy threshold for base64 strings
                        [default: 4.5]
  --entropy-hex-thresh ENTROPY_HEX_THRESH
                        User defined entropy threshold for hex strings
                        [default: 3.0]
  --since-commit SINCE_COMMIT
                        Only scan from a given commit hash
  --max-depth MAX_DEPTH
                        The max commit depth to go back when searching for
                        secrets
  --branch BRANCH       Name of the branch to be scanned
  --repo-path REPO_PATH
                        Path to the cloned repo. If provided, git_url will not
                        be used
  --cleanup             Clean up all temporary result files
```

## git-forest
Along with `foresthog` utility that analysis the static code of your committed phase, another handy tool `git-forest` is added to integrate this code analysis with git hooks.
### Initiate Forest Integration
You can start the integration with following command. By default, `post-commit` and `pre-push` events will be integrated.
```
git-forest init
```
After initialization, you can change the config of your `foresthog` analysis using `git-forest update` sub-utility or by directly editing the file: `<GIT_ROOT>/.forest/config.json`.  
A sample of config, in JSON, is as follows:
```
{
  "custom_rules": {},
  "entropy_wc": 20,
  "entropy_hex_thresh": 3.0,
  "entropy_b64_thresh": 4.5,
  "enable_default_rules": true,
  "enable_regex": true,
  "enable_entropy": true,
  "pre_push": true,
  "post_commit": true
}
```

### Update Forest Configuration
You can update them either manually or using `update` command. For example, to disable post-commit trigger:
```
git-forest update --no-post-commit
```
Similarly, you can change the entropy thresholds like:
```
git-forest update -entropy-hex-thresh 3.3
```
For `custom-rules`, you need to provide a JSON file that contain those rules or manually insert them in config file.
```
git-forest update -custom-rules <filename.json>
```  
> Git hooks make use of `git-forest run` command and each event triggers a call which reads the config file on the go and analyse the code using `foresthog` utility and passes / fails the code as return value.  

  

### Destroy Forest Integration
To remove the integration, run the following command.
```
git-forest destroy
```
It will remove `.forest` directory from `<GIT_ROOT>` and git hooks are disconnected and backed up with `.bkp` extention in `<GIT_ROOT>/.git/hooks` directory.

### Help Dialog
```
usage: git-forest [-h] [-trigger {pre-push,post-commit}]
                  [-custom-rules CUSTOM_RULES] [-entropy-wc ENTROPY_WC]
                  [-entropy-hex-thresh ENTROPY_HEX_THRESH]
                  [-entropy-b64-thresh ENTROPY_B64_THRESH] [--no-regex]
                  [--no-default-rules] [--no-entropy] [--no-pre-push]
                  [--no-post-commit]
                  {init,run,update,destroy}

positional arguments:
  {init,run,update,destroy}
                        defines foresthog operations.

optional arguments:
  -h, --help            show this help message and exit
  -trigger {pre-push,post-commit}
                        runs forestHog for certain git event.
  -custom-rules CUSTOM_RULES
                        custom rules files to check the git repo against.
  -entropy-wc ENTROPY_WC
                        sets string length for calculating entropy.
  -entropy-hex-thresh ENTROPY_HEX_THRESH
                        sets entropy threshold for hex strings.
  -entropy-b64-thresh ENTROPY_B64_THRESH
                        sets entropy threshold for base64 strings.
  --no-regex            disables regex checks for git repo.
  --no-default-rules    disables default rule checks for git repo.
  --no-entropy          disables entropy checks for git repo.
  --no-pre-push         disables pre-push hook to git repo.
  --no-post-commit      disables post-commit hook to git repo.
```

### Things to Keep in Mind
1. `post-commit` hook should always be enabled. It will help you figure out which commit failed the code analysis and revert the commit right there. Using only `pre-push` still prevents your code to go online but you might have to revert multiple commits to get rid of sensitive data.
2. Use custom rules, along with default ones, to cater to your specific needs. Default rules cover most widely used key patterns but it may not necessary fulfill what you actually need. If default rules are too restrictive, you can customize to ignore them and use only your custom rules / regexes.
3. Entropy is helpful to get noisy strings from the code which can potentially be a critical information in your code. You might need to tweak the threshold values to your needs.

> For organizations, it's a best practice to regularly review their public repo for leakage of critical information. `foresthog` utility can easily analyse an online repo.

## Reverting Git Commits
To revert a last commit without losing any changes on files, it's best to use:
```
git reset HEAD~
git reset --hard HEAD~ # changes will vanish as well.
```
To revert changes on a file to last committed state, you can use:
```
git checkout <filename/wildcard>
```
To revert last push from online repo, use the following:
```
git push <remote> +HEAD^:<branch>
git push origin +HEAD^:master
```
To filter out entire git tree, you can use:
```
git filter-branch --tree-filter <command>
git filter-branch --tree-filter "rm -f <sensitive-filename.txt>"
```
Check out the documentation to learn more about this amazing utility: [filter-branch](https://git-scm.com/docs/git-filter-branch)  
Another great tool to rewrite history is [bfg](https://github.com/rtyley/bfg-repo-cleaner). By giving it a file having regex of things you want to replace, it will spread the effect to entire git history.

### Futility of Rewrites
If your repo had been public, there is a chance of it being cloned by someone and your critical data being disclosed, if there was any. Rewriting history / commits will prevent any leaking in future but you can't do anything about data leaked with cloned repos.  
Rewriting history may sound cool but if done carelessly, it can mess up your entire git tree structure making it unable to be used any further.
>*Use it with great care and use it only as a last resort.*

Lastly, rewriting history means you're making a change in a branch. You will have to notify your coworkers to have them clean their branches as well to make a seemless git structure. Otherwise, they will face errors upon merging and pushing code.