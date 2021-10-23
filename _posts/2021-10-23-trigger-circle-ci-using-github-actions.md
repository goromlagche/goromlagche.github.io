---
layout: posts
title:  "Trigger circle-ci using github actions [Part 4]"
excerpt: Slash test suite running time by using circle-ci parallel runs, splitting the test suite by timing data. Add crystalball magic to further boost the test run time
date:   2021-10-23 18:02:47 +0530
layout: single
tags: [rspec, rails, crystalball, github-actions, circle-ci, ruby, crystalball-circle-ci-series, rake]
---
We will explore how to optimize your test suite using [Crystalball](https://github.com/toptal/crystalball) test selection library, to reduce the test run time, down to a minute. This blog post is an attempt to document my own experience of setting up crystalball alongside circle-ci parallel runs.

This is **Part 4** of this series.

You can find all the links to this series [here]({{ site.baseurl }}/tags/#crystalball-circle-ci-series)

## Using Github actions

So far we have done all the setup for crystalball to run for a non-master branch.

A developer can branch off from master, and push their commits regularly to the origin. And every time CI will run the minimal number of tests required.

But once the developer is done with their work, and ready to open a pull request for a code-review. Ideally, we should run the entire test-suite once.

This would achieve broadly two things

1. Be double sure that no regression has happened
2. If you have code-coverage setup. Sometimes there are checks like test coverage should be > 98%. It is difficult to ascertain that without actually running the entire test suite.

So, when a developer is ready for code review, they can add a github label `ready-for-code-review`. We can use github actions to trigger a full circle-ci run.

Sounds like a plan?!

Let's implement that.

## Store circle ci token in Github secret

For this, first, you will need to create an action secret. You can read about this more [here](https://docs.github.com/en/actions/security-guides/encrypted-secrets).

In a github secret you will need to store the `CIRCLE_CI_TOKEN`, created all the way back in [Part-2]({{ site.baseurl }}{% link _posts/2021-09-26-storing-crystalball_data-as-a-circle-ci-artifact.md %}#storing-crystalball-data).

Once this is setup, you should be able to access the circle-ci token using `secrets.CIRCLE_CI_TOKEN`

## Lights, camera, and action!

Now that we have the secrets setup, we can actually create the github action.

{% raw  %}
``` yaml
name: Run Circle-CI on Label
on:
  pull_request:
   types:
    - labeled
jobs:
  ready_for_code_review_job:
    if: ${{ github.event.label.name == 'ready-for-code-review'  }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: run full spec suite
        id: run-rspec-in-circle-ci
        run: |
          curl --request POST \
               --url https://circleci.com/api/v2/project/gh/your_github_org/your_github_repo/pipeline \
               --header 'Circle-Token: ${{ secrets.CIRCLE_CI_TOKEN }}' \
               --header 'content-type: application/json' \
               --data '{"branch":"${{ github.event.pull_request.head.ref }}","parameters":{"crystalball": false}}'
```
{% endraw %}

Setting `crystalball` param to `false` in circle-ci will make sure that the entire test suite is run.

Store this in `.github/workflows/labels.yml` file. And it should be good to go.

## The End!
Alright, finally we are done!

I generally do not write long-form blog posts. I tend to struggle with them. But, I had a difficult time setting up crystalball and make it play nice with circle-ci. These posts are written to help anyone who might face similar issues.

Until next week! :heart:
