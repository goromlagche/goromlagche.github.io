---
layout: posts
title:  "Storing crystalball_data as a circle-ci artifact [Part 2]"
excerpt: Slash test suite running time by using circle-ci parallel runs, splitting the test suite by timing data. Add crystalball magic to further boost the test run time
date:   2021-09-26 19:02:47 +0530
layout: single
tags: [rspec, rails, crystalball, circle-ci, ruby, crystalball-circle-ci-series, rake]
---
We will explore how to optimize your test suite using [Crystalball](https://github.com/toptal/crystalball) test selection library, to reduce the test run time, down to a minute. This blog post is an attempt to document my own experience of setting up crystalball alongside circle-ci parallel runs.

This is **Part 2** of this series.

You can find all the links to this series [here]({{ site.baseurl }}/tags/#crystalball-circle-ci-series)

## Storing crystalball data

Every time crystalball runs, it will generate a `crystalball_data.yml` file, which stores the mapping between source code and test files. This is the source of truth, using which crystalball predicts the minimal subset of tests needed to be run.

Our strategy will be to run a full test suite when the branch is master. And store the `crystalball_data.yml` files in a circle-ci artifact.

When the branch is not master, we will download the `crystalball_data.yml` files artifact and leverage crystalball to predict the minimum number of tests required to run.

Let us extend our circle-config file from [Part-1]({{ site.baseurl }}{% link _posts/2021-09-18-supercharge-rspec-with-crystalball.md %}#setup-circle-ci-config).

``` yaml
version: 2.1
executors:
  rspec-executor:
    environment:
      CRYSTALBALL: true
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
      - run:
          name: Stash crystalball results
          command: |
            if [[ "$CRYSTALBALL" == "true" ]]; then
              mkdir -p crystalball
              cp -R ~/tmp/crystalball_data.yml ~/crystalball/crystalball_data-${CIRCLE_NODE_INDEX}.yml
            end
      - store_artifacts:
          path: ~/crystalball
```

This will store the crystalball map data in a circle-ci artifact.

Once we have been able to persist all the data in a circle-ci artifact, we should be able to download the same. We will need to use circle-ci's API to achieve that.

Time to hit the [docs](https://circleci.com/docs/api/v2/). We will be using the latest V2 version of the API.

To use the API, first, you will need to [Create a personal API token ](https://circleci.com/docs/2.0/managing-api-tokens/#creating-a-personal-api-token). You can set an env variable `CIRCLE_CI_TOKEN` which will store the same.

[This]((https://circleci.com/docs/api/v2/#operation/getJobArtifacts)) is the particular API that allows us to download an artifact.

To download an artifact, it is important to fetch the `job-number`. It is the identifier of the job which stores all the circle-ci artifacts.

Inorder to achive this,
* Create a new job `store_crystalball_build_num`
* Use `CIRCLE_BUILD_NUM` which is an env variable provided by circle-ci, for each running job.
* Create a new env variable `CRYSTALBALL_BUILD_NUM` which will store `CIRCLE_BUILD_NUM` for the `store_crystalball_build_num` job.
* Use this [API](https://circleci.com/docs/api/v2/#operation/createEnvVar) to create and persist our new env variable `CRYSTALBALL_BUILD_NUM`
* Now, from any job, we can use `CRYSTALBALL_BUILD_NUM` to download all the artifacts

Let us modify the circle-ci config file to achieve this.

``` yaml
version: 2.1
executors:
  rspec-executor:
    environment:
      CRYSTALBALL: true
   ...
   ...
jobs:
  rspec:
    executor: rspec-executor
    parallelism: 5
    steps:
      - run: bundle exec rails db:reset
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
      - run:
          name: Stash crystalball results
          command: |
            if [[ -e ~/tmp/crystalball_data.yml ]]; then
              mkdir -p crystalball
              cp -R ~/tmp/crystalball_data.yml ~/crystalball/crystalball_data-${CIRCLE_NODE_INDEX}.yml
            end
  store_crystalball_build_num:
    executor: rspec-executor
    steps:
      - run:
          name: Store crystalball build num
          command: |
            bundle exec rails ci:store_crystalball_build_num
      - store_artifacts:
          path: ~/crystalball
workflows:
  ci:
    - rspec:
      ...
      ...
    - store_crystalball_build_num:
          requires:
            - rspec
          filters:
            branches:
              only:
                - master
```

We will create a rake task `ci:store_crystalball_build_num` and a small service `CrystalballCiService` to do the heavy lifting

``` ruby
namespace :ci do
  desc 'store crystalball build number, we need this to retrieve all the artifacts'
  task store_crystalball_build_num: :environment do
    return if ENV['CRYSTALBALL'] != 'true'

    crystalball_ci_service = CrystalballCiService.new
    crystalball_ci_service.store_crystalball_build_num
  end
end
```

``` ruby
class CrystalballCiService
  CRYSTALBALL_DIR_PATH  = Rails.root.join('crystalball')

  def initialize
    @base_url = "https://circleci.com/api/v2/project/gh/"\
                "#{your_github_org}/#{your_github_repo}"
  end

  def store_crystalball_build_num
    body = { name: 'CRYSTALBALL_BUILD_NUM',
             value: ENV['CIRCLE_BUILD_NUM'] }
    circleci_request(url: "#{@base_url}/envvar", method: :post, body: body)
  end

  private

  def circleci_request(url:, method:, body: nil, opts: {})
    RestClient::Request.execute(
      method: method,
      url: url,
      headers: { 'Circle-Token' => ENV['CIRCLE_CI_TOKEN'] },
      payload: body,
      **opts
    )
  end
end
```

That's it!

Now we can
1. Run entire spec suite in master branch.
2. Store the crystalball mapping files in a circle-ci artifact.
3. Store the build number of the job which pushes the map files to an artifact.

Next in [Part-3]({{ site.baseurl }}{% link _posts/2021-10-16-download-artifacts-from-circle-ci.md %}) I will discuss retrieving crystalball map files and use the same to run a minimal number of tests required for a non-master branch.

Stay tuned! :heart:
