# The JAM stack - GitHub - Proflow


## To install
* Copy the `.circleci` folder in the root of your repo

An example using `curl`:
```bash
# cd to the root of your repo and then run
TMP_SRC=https://raw.githubusercontent.com/prolike/circleci-templates/master/proflow/github/jam/.circleci
mkdir .circleci && cd $_
curl -O $TMP_SRC/config.yml
curl -O  $TMP_SRC/play-proflow-cci-gh.yml
TMP_SRC= # cleaning up
```

...hmm, that's it!


**The workflow supports 4 different flows:**

1. [A dev branch](#a-development-branch---the-basic-flow)
2. [A `ready/*` branch (created using the Phlow)](#a-ready-branch)
3. [The `master` branch](#the-master-branch)
4. [Annotated tags](#annotated-semver-tags)

It's designed to build a [JAM stack](http://jamstack.org) - we use Jekyll, but it could be any. We're using GitHub as host for `origin`. In a JAM stack the final output of the build process is a static web site, in our flow it's deployed to GitHub Pages, but you can easily alter it to what ever you like - the four different flows are conceptual, they should work for any development setup.

The _play book_  `play-proflow-cci-gh.yml` represents a configuration approach to running shell commands. I've added a elaborated discussion on the `play` script in [the end of this doc](#a-few-comments-on-the-play-book).


# A development branch - the basic flow
The flow allows you to push _any_ branch you desire and that way use the Circlel CI setup as a _build resource_. It will not deliver or deploy anything, it will just run your _Definition of Done_.

This basic flow would be triggered by something like this:

``` shell
git checkout -b some-branch
# hack hack
git commit -am "let's see the circle ci basic flow run"
git push
```

![Dev-branch](images/dev-branch.png)˛

The fist step `prep-repo` simply prepares the repo - the step only has a simple function in this flow; it creates a Circle CI cache, so in the following steps we'll save time by reusing the cache, rather than pulling it from GitHub in every step.

The second step `jekyll-build` builds the static web site and then saves it in another Circle CI cache - containing only the derived objects form the build process.

This is probably the step you want to change, if you are using a different JAM stack.

The test are there to show, that at this point it's possible to run tests in parallel - they all operate on the static site data - and not the actual git repo.

And that's really it - in this flow.

**Pro tip:**
If you only created this commit to trigger Circle CI to build, and you really would like to _"dissolve"_ it back to just edited files in you index then an `undo-commit` command would be handy:

```bash
git config --global alias.undo-commit 'reset --soft HEAD^' #create the alias in your personal .gitconfig
git undo-commit
```

# A ready-branch

When a branch arrive that has the prefix `ready/` it triggers the flow to integrate it in to the target branch - most often `master`.

The flow would be triggered by something like this:

``` shell
git checkout -b some-branch
# hack hack
git commit -am "let's use circle ci 'deliver' to integrate this branch into master"
git push some-branch:ready/some-branch
```

![ready branch](images/ready-branch.png)

It's essentially the same flow as the basis flow above, but with two differences. In the `prep-repo` the ready branch is rebased to `master` and then merged into it - but only if the merge is fast-forward. Effectively Circle CI will refuse to create a new commit - silently assuking that the rebase wasn't needed - if it was, you should have done it locally before your push.

The `ready/*` prefir to the branch also triggers a final step `deliver` which pushes master back to GitHub.

Ideally you could then put a [restriction on the target integration branch](https://help.github.com/en/articles/enabling-branch-restrictions) (probably `master`), So that only the _release_ user running the Circle CI flow can push to it.

# The master branch

The master branch flow is really the same as the basis flow, only exception is that on successful execution it' ends with a deploy to the stage environment.

The idea is that the `master` branch is a genuine _release train_. In a perfect world, there's no way you can trigger this directly. The flow is that you should get your stuff onto the `master`branch using the `ready/*` flow mentioned above, and if successfull it will _automatically_ trigger this flow.


![stage deploy](images/master-branch.png).

It simply pushes the static site data to a dedicated GitHub repo, which is setup to be served by GitHub Pages.

# Annotated SemVer tags

The flow is identical to the one for the master branch, except the final step pushes to a different target and it's triggered by a tag that complies to the [_semantic versioning_](http://www.semver.org), but only if the tag is annotated.

If you ran the following:

``` bash
git tag 1.0.1
git tag -m "Release feature xyz" 1.0.2
git push --tags
```

Then the flow will run twice, once for each tags, but the flow for tag `1.0.1` will fail because it's not an annotated tag.

A tag that isn't annotated is essentially just a pointer to an existing commit, while an annotated tag is a first-class citizen in git, with it's own integrity.


![semver-tag](images/semver-tag.png)

# A few comments on the play book

The deliver step that wraps up the flow when triggered by `master` serves as a fine example, as it's really short.

In the `config.yml` file you have:

```yaml
deliver:
  working_directory: /app
  docker:
    - image: lakruzz/play:latest
  steps:
    - restore_cache:
        keys:
          - the-repo-{{ .Revision }}
    - run:
        name: Deliver
        command: play --manuscript .circleci/play-proflow-cci-gh.yml --part deliver
```

This makes it nice and easy to read in the `config.yml`. The `play-proflow-cci-gh.yml` is the "play-book". It uses the analogy of a "play" performed at a "theater" it's also laid out as nice readable yaml, and defines different "props" (environment variables) end the "scene"  which can be more or less verbose and then it defines the "parts" which all have "lines" to perform.

Let's look at the `--deliver` part in the  `.circleci/play-proflow-cci-gh.yml` invoked from the `config.yml` above:

```yaml
manuscript:
  scene:
    dryrun:  no
    debug:   no
    verbose: no
  parts:
    deliver:
      props:
        - PLAY_REPOSITORY: '`printenv CIRCLE_REPOSITORY_URL | sed  s/".*github.com[\/:]//" | sed s/.git//`'
      scene:
        verbose: yes
      lines:
        - run:
            caption: Add a remote that has permissions to write
            command: git remote add integrated https://$PHLOW_GHTOKEN@github.com/$PLAY_REPOSITORY.git
            die_on_err: yes
        - run:
            caption: Push current branch to GitHub
            command: git push integrated
            die_on_err: yes
        - run:
            caption: Delete the remote triggering branch
            command: git push integrated :$CIRCLE_BRANCH
            die_on_err: yes
```

It's obvious that this part is supposed to run in Circle CI context, as it assumes the existence of both `$CIRCLE_REPOSITORY_URL` and `$CIRCLE_BRANCH`. The props are just a nickname for the environment variables to set. The values however can be executable strings.

The deliver part also mentions the environment variable `$PHLOW_GHTOKEN`. This is a GitHub Developer [personal access token](https://github.com/settings/tokens) with write access to the repos. It is stored in the environment variable `$PHLOW_TOKEN` set in the Circle CI settings.
