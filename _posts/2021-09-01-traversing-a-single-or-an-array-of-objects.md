---
layout: posts
title:  "Traversing a single or an array of objects"
date:   2021-09-01 16:57:47 +0530
layout: single
author_profile: true
comments: true
tags: [ruby, refactoring, xml]
---
I will explain a simple pattern to process a single or an array of objects uniformly. Since ruby is not a strongly typed language(< ruby 3), the burden may lie on the input function.

I will start with an example.

Let say we need to parse an XML schema. Which may or may not have a repeating block.

Example 1
``` xml
<?xml version="1.0" encoding="UTF-8"?>
<comments>
    <comment>
        <description>TEST COMMENT 123</description>
    </comment>
</comments>
```

Example 2
``` xml
<?xml version="1.0" encoding="UTF-8"?>
<comments>
    <comment>
        <description>TEST COMMENT 123</description>
    </comment>
    <comment>
        <description>TEST COMMENT 456</description>
    </comment>
</comments>
```

First lets write a function to parse the data. I will be using [nori](https://github.com/savonrb/nori)

``` ruby
require 'nori'

def fetch_descriptions_1(raw_data:)
  comments = Nori.new.parse(raw_data)['comments']['comment']
end
```

Next, let us try to fetch the descriptions. Since comment can be both a `single object` or an `array of objects`. We can use a simple if-else clause for this.

``` ruby
def fetch_descriptions_1(raw_data:)
  comments = Nori.new.parse(raw_data)['comments']['comment']
  if comments.is_a?(Array)
    comments.map { |comment| comment['description'] }
  else
    [comments['description']]
  end
end
```

The above perfectly handles the volatile nature of the input data.

But we can do this in a better manner. There is an opportunity to refactor.

Consider this, if we put the single_comment inside an array, then we will only need to handle a single input type, `array of objects`.

``` ruby
def fetch_descriptions_2(raw_data:)
  comments = [Nori.new.parse(raw_data)['comments']['comment']].flatten
  comments.map { |comment| comment['description'] }
end
```


Full example

``` ruby
require 'nori'

single_comment = <<~EOF
<?xml version="1.0" encoding="UTF-8"?>
<comments>
    <comment>
        <description>TEST COMMENT 123</description>
    </comment>
</comments>
EOF

multiple_comments = <<~EOF
<?xml version="1.0" encoding="UTF-8"?>
<comments>
    <comment>
        <description>TEST COMMENT 123</description>
    </comment>
    <comment>
        <description>TEST COMMENT 456</description>
    </comment>
</comments>
EOF

def fetch_descriptions_1(raw_data:)
  comments = Nori.new.parse(raw_data)['comments']['comment']
  if comments.is_a?(Array)
    comments.map { |comment| comment['description'] }
  else
    [comments['description']]
  end
end

def fetch_descriptions_2(raw_data:)
  comments = [Nori.new.parse(raw_data)['comments']['comment']].flatten
  comments.map { |comment| comment['description'] }
end
```
