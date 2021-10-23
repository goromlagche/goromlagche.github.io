---
layout: posts
title:  "Download artifacts from circle-ci [Part 3]"
excerpt: Slash test suite running time by using circle-ci parallel runs, splitting the test suite by timing data. Add crystalball magic to further boost the test run time
date:   2021-10-16 19:02:47 +0530
layout: single
tags: [rspec, rails, crystalball, circle-ci, ruby, crystalball-circle-ci-series, rake]
---
We will explore how to optimize your test suite using [Crystalball](https://github.com/toptal/crystalball) test selection library, to reduce the test run time, down to a minute. This blog post is an attempt to document my own experience of setting up crystalball alongside circle-ci parallel runs.

This is **Part 3** of this series.

You can find all the links to this series [here]({{ site.baseurl }}/tags/#crystalball-circle-ci-series)

## Download crystalball data

Generally to use `crystalball`, all you need to do is run `bundle exec crystalball`. Crystalball has a custom RSpec runner. It builds a prediction and runs it.

But we want to leverage `circle-ci`'s ability to do test splitting based on timing data. So we can not use `crystalball` rspec runner as is.

We will use `crystalball` to build the prediction, generate the minimal number of spec files that need to be run. And then pass that list of files to `circle-ci` so that it can split the files based on timing data and parallelize the test run across multiple containers.

To do so, we can write a rake task to generate the minimal subset of test needed to be run

``` shell
TESTFILES=$(bundle exec rake circleci:spec_files | circleci tests split --split-by=timings)
```

Let us extend our circle-config file from [Part-2]({{ site.baseurl }}{% link _posts/2021-09-26-storing-crystalball_data-as-a-circle-ci-artifact.md %}#storing-crystalball-data).

``` yaml
jobs:
  rspec:
    executor: rspec-executor
    parallelism: 3
    steps:
      - run:
          name: Run rspec in parallel
          command: |
            bundle exec rake circleci:spec_files > /tmp/spec.files
            if [ -s /tmp/spec.files ]; then
              TESTFILES=$(circleci tests split --split-by=timings --show-counts /tmp/spec.files)
              echo $TESTFILES
              [[ ! -z "${TESTFILES}" ]] && bundle exec rspec --format progress \
                                --no-color \
                                --format RspecJunitFormatter \
                                --out ~/test-results/rspec/rspec.xml \
                                ${TESTFILES}
            fi
```

Let us write the rake task to run the crystalball predictor and generate the test files.

``` ruby
namespace :ci do
  desc 'List spec files'
  task spec_files: :environment do
    original_stdout = $stdout.clone
    $stdout.reopen File.new('/dev/null', 'w')

    _error_stream = StringIO.new
    output_stream = StringIO.new

    crystalball_ci_service = CrystalballCiService.new
    files = if crystalball_ci_service.runner == 'crystalball'
              Crystalball::RSpec::Runner.run(['--dry-run', '--format=json'],
                                             _error_stream,
                                             output_stream)
              data = JSON.parse(output_stream.string)
              data['examples'].map do |example|
                example['id'].split('[').first.split('./').last
              end.uniq
            else
              Dir.glob('spec/**/*_spec.rb')
            end

    original_stdout.write files.join("\n")
  end
end
```

All this task does is
1. checks the current branch.
2. if it is master, returns all the test files.
3. if it is not master, we are using `Crystalball::RSpec::Runner.run` to generate the test files.

Since this rake task is piping the output to `circleci` cli, it is important to keep the stdout in check.

Ok now we have this setup, next up we have to tackle the most important thing. Retrieving `crystalball_data.yml` files which store the mapped data between tests and the application code.

To do this, we will be using the env variable `CRYSTALBALL_BUILD_NUM` stored on the job `store_crystalball_build_num` discussed in [Part-2]({{ site.baseurl }}{% link _posts/2021-09-26-storing-crystalball_data-as-a-circle-ci-artifact.md %}).

Let's look at the code.

``` ruby
class CrystalballCiService
  CRYSTALBALL_FILE_PATH = 'crystalball/crystalball_data'.freeze
  CRYSTALBALL_DIR_PATH  = Rails.root.join('crystalball')

  def initialize
    @base_url = "https://circleci.com/api/v2/project/gh/"\
                "#{your_github_org}/#{your_github_repo}"
  end

  def download_crystalball_file
    artifacts = circleci_request(url: "#{@base_url}/#{ENV['CRYSTALBALL_BUILD_NUM']}/artifacts",
                                 method: :get)

    files = JSON
              .parse(artifacts.body)['items']
              .select { |artifact| artifact['path'].include? CRYSTALBALL_FILE_PATH }

    Dir.mkdir(CRYSTALBALL_DIR_PATH) unless File.exist?(CRYSTALBALL_DIR_PATH)

    files.each do |file|
      raw = circleci_request(url: CGI.unescape(file['url']),
                             method: :get,
                             opts: { raw_response: true })
      File.rename(raw.file.path, File.expand_path(file['path']))
    end
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

And that is it!

Now we have enough code to
1. Run entire spec suite in master branch.
2. Run only the minimal set of tests required for a non-master branch.

In the next part [Part-4]({{ site.baseurl }}{% link _posts/2021-10-23-trigger-circle-ci-using-github-actions.md %}), which will be a bonus post. I will discuss why and how we should/can run the entire test suite once a PR is ready for code review.

Stay tuned! :heart:
