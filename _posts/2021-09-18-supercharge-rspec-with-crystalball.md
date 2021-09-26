---
layout: posts
title:  "Supercharge tests with circle-ci & crystalball [Part 1]"
excerpt: Slash test suite running time by using circle-ci parallel runs, splitting the test suite by timing data. Add crystalball magic to further boost the test run time
redirect_from:
  - /supercharge-rspec-with-crystalball
  - /supercharge-rspec-with-crystalball/
date:   2021-09-18 17:18:47 +0530
layout: single
tags: [rspec, rails, crystalball, circle-ci, ruby, crystalball-circle-ci-series]
---
We will explore how to optimize your test suite using [Crystalball](https://github.com/toptal/crystalball) test selection library, to reduce the test run time, down to a minute. This blog post is an attempt to document my own experience of setting up crystalball alongside circle-ci parallel runs.

This is **Part 1** of this series.

You can find all the links to this series [here]({{ site.baseurl }}/tags/#crystalball-circle-ci-series)

## Background

For large codebases, handling a huge test suite becomes challenging soon.

* The journey of a test suite starts small, finishing up within a couple of minutes in a local machine.
* Gradually the codebase starts to gain some mileage. Rspec runs notching > 10 mins in the CI.
* Flaky tests start to show up. CI runs retried multiple times. The alarm bell starts ringing. Sounds familiar?!

One of the approaches to solve this problem is to use predictive test-selection.
Thankfully in ruby world, there is already a ready-made solution available.

**Aaron Patterson** has written about [this](https://tenderlovemaking.com/2015/02/13/predicting-test-failues.html), and the good folks over toptal have built the library [Crystalball](https://github.com/toptal/crystalball) :heart:.

### Here are some references to get to know about crystalball

1. The [documentation](https://toptal.github.io/crystalball/) is pretty detailed.
2. This [presentation](https://rubykaigi.org/2019/presentations/p0deje.html) at ruby kaigi is a great demonstration of the features.

## Setup Circle-CI config

Circle-CI provides parallel runs based on automatic test splitting based on multiple strategies. Their [documentation](https://circleci.com/docs/2.0/parallelism-faster-jobs/) provides details on this.

IMO the most useful test-splitting strategy is `--split-by=timings`.

We can set-up parallel runs in the circle-ci config.

``` yaml
version: 2.1
executors:
  rspec-executor:
   ...
   ...
jobs:
  rspec:
    executor: rspec-executor
    parallelism: 5
    steps:
      - run:
            name: Run rspec in parallel
            command: |
              TESTFILES=$(circleci tests glob "spec/**/*_spec.rb" | circleci tests split --split-by=timings)
              bundle exec rspec --format progress \
                                --format RspecJunitFormatter \
                                --out ~/test-results/rspec/rspec.xml \
                                ${TESTFILES}
      - store_test_results:
          path: ~/test-results

workflows:
  ci:
    - rspec:
      ...
      ...
```


## Setup Crystalball

Out in the wild, one of the prominent OSS codebase, which uses **Crystalball** is **Gitlab**. They have documented their usage of `crystalball` in their CI flow [here](https://docs.gitlab.com/ee/development/pipelines.html#rspec-minimal-jobs).

IMO that is a good direction to follow. Using that we can come up with a workflow. :point_down:

### Workflow Diagram

![development-workflow](/assets/images/development-workflow.png)

To setup crystalball, first, we will need to install the gem

``` ruby
gem install crystalball
```

Since we are running our specs in parallel, multiple mapping files will be generated. Hence it is important to store all of them in a folder. That can be defined in `config/crystalball.yml` file.

``` yaml
# Default: `tmp/crystalball_data.yml`
execution_map_path: crystalball
```

In the `spec/spec_helper.rb`, we will define all the strategies

``` ruby
if ENV['CRYSTALBALL'] == 'true'
  require 'crystalball'
  require 'crystalball/rails'

  Crystalball::MapGenerator.start! do |config|
    config.register Crystalball::MapGenerator::CoverageStrategy.new
    config.register Crystalball::MapGenerator::AllocatedObjectsStrategy.build(only: ['Object'])
    config.register Crystalball::MapGenerator::DescribedClassStrategy.new
    config.register Crystalball::Rails::MapGenerator::I18nStrategy.new
    config.register Crystalball::Rails::MapGenerator::ActionViewStrategy.new
  end
end
```

Once all of this is done, we will need to make **Crystalball** play nice with **Circle-CI**.

That will be a code heavy post, which I will take up in [Part-2]({{ site.baseurl }}{% link _posts/2021-09-26-storing-crystalball_data-as-a-circle-ci-artifact.md %}) of this series.
