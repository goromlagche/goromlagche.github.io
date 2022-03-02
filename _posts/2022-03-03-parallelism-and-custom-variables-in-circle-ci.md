---
layout: posts
title:  "Parallelism and custom variables in CircleCI "
excerpt: Using custom variables to make parallelism configurable in CircleCI
date:   2022-03-03 01:44:47 +0530
layout: single
tags: [circle_ci, rspec]
---
CircleCI allows us to specify a job's parallelism level. In this post, we will try to further customize that using a custom CircleCI parameter.

## Premise
We can use CircleCI's [parallelism](https://circleci.com/docs/2.0/parallelism-faster-jobs/) feature to split a job in parallel by spreading them across multiple separate executors.

But sometimes we want to customize the parallelism count based on various other parameters.

One example of this would be, let's say we have two kinds of workflows. A `ci-workflow` that runs every time. And a `nightly` workflow that runs at a certain time every night and builds the nightly release.

``` yaml
jobs:
  rspec:
    executor: rspec-executor
    parallelism: 30
    steps: ...

workflows:
  nightly:
    triggers:
      - schedule:
          cron: "00 10 * * *"
          filters:
            branches:
              only:
                - master

    jobs:
      - rspec:
          requires:
            - bundle-install

  ci-workflow:
    jobs:
      - rspec:
          requires:
            - bundle-install
```

We are using parallelism of 30 for our RSpec tests. For both the workflows. Now `nightly` runs happen at a low traffic time when most of the devs are asleep. We can reduce the number of parallelism for just the nightly workflow.

## Solution
CircleCI does not provide an out-of-the-box solution for the above problem. We will need to rely on a custom parameter to solve this problem.

``` yaml
jobs:
  rspec:
    executor: rspec-executor
    parameters:
      machines:
        type: integer
        default: 30
    parallelism: << parameters.machines >>
    steps: ...

workflows:
  nightly:
    triggers:
      - schedule:
          cron: "00 10 * * *"
          filters:
            branches:
              only:
                - master

    jobs:
      - rspec:
          requires:
            - bundle-install

  ci-workflow:
    jobs:
      - rspec:
        machines: 10
          requires:
            - bundle-install
```

Using a pipeline parameter like above we can customize the parallelism value in CircleCI.

Until next time! :heart:
