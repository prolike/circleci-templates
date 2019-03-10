# The changeset trigger


## To install
* Copy the `.circleci` folder in the root of your repo

An example using `curl`:
```bash
# cd to the root of your repo and then run
TMP_SRC=https://raw.githubusercontent.com/prolike/circleci-templates/master/changeset-trigger/.circleci
mkdir .circleci && cd $_
curl -O $TMP_SRC/config.yml
cd .. && TMP_SRC= # cleaning up
```

...hmm, that's it!


**The workflow enables you to do something, only when certain files of folcers are changed**

Imagine that you have a folder in your repo named `.github` in which you have some config files
that defines exactly how you want this repo to look like on GitHub, could be that you have a decription of the default labels to use with your issues: `.github/labels.yml`.

Let's say that:

>"If this this file changes on the master branch, then Circle CI should run a script, that updates the labels on the repository on GitHub"

The ultra short version of a `.circleci/config.yml` that could accoplish something like that looks like this:

```yaml
jobs:
  path-trigger:
    working_directory: /app
    docker:
      - image: lakruzz/play:latest
        environment:
          CIRCLE_PLAY_PATH_REGEXP: .github/labels.yml
    steps:
      - checkout
      - run:
          name: Run if we should
          command: |
            export PLAY_RUN=$(git diff-tree  --no-commit-id --name-only -r $CIRCLE_SHA1 \
              | grep -e "$CIRCLE_PLAY_PATH_REGEXP")
            [[ -z "$PLAY_RUN" ]] \
              && echo "Not my call!" \
              || echo "Yeeah! We have work to do, run your script here!"
workflows:
  version: 2
  Proflow:
    jobs:
      - path-trigger:
          filters:
            branches:
              only: /^master$/
```

But explore the actual [`.config.yml`](../config.yml), it's a bit more elaborate, and explains some of the nitty gritty details.
