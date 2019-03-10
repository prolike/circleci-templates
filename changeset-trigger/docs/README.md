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

version: 2
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
          name: Check the changeset for a match in $CIRCLE_PLAY_PATH_REGEXP
          command: |
            if [[ \
              $(git diff-tree  --no-commit-id --name-only -r `git rev-parse --short HEAD` \
              | grep -e "$CIRCLE_PLAY_PATH_REGEXP") ]];\
            then \
              echo "Yeeah! We have work to do, run your script here"; \
            else \
              echo "Not my call\!";\
            fi;

workflows:
  version: 2
  Proflow:
    jobs:
      - path-trigger:
          filters:
            branches:
              only: /^master$/

```

But explore the actual [`.config.yml`](../config.yml), it's a bit more elaborate.
