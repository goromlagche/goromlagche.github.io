---
layout: posts
title:  "Implementing cursor-based pagination"
excerpt: We will implement a cursor-based pagination module, also discuss about some gotchas for handling UUIDs
date:   2022-01-26 18:49:47 +0530
layout: single
tags: [database, postgres, rails, active_record, pagination]
---
Pagination is ubiquitous while working with web applications. Although it is simple enough with limit and offset queries. Performance becomes a problem once you hit a certain scale.

## Premise

[Kaminari](https://github.com/kaminari/kaminari) is a very popular and an awesome pagination library in the ruby world. It is very simple to use and integrate it into your project.

But as soon as your project starts to get some traction and reaches some amount of scale, you will start to notice certain issues.

For example, the kaminari default setup will run at least two queries,
1. `select * from table limit x offset y`
2. `select count(*) from table`

The second query is needed to know the total number of records to display the links. But once your table has enough records, after a certain point, you will notice the `count` query takes more time than the first pagination query.

Kaminari provides a way to handle the issue in [here](https://github.com/kaminari/kaminari#paginating-without-issuing-select-count-query). Awesome!

Once your project becomes even more popular. And you have 100s of pages on your web app. You will notice a clear drop in response time when browsing further back.

This is an awesome [blog post](https://use-the-index-luke.com/sql/partial-results/fetch-next-page) by **Markus Winand** explaining the issues of using offset and why we should consider moving to cursor-based pagination, once we hit the scale. There is also a [slide deck](https://use-the-index-luke.com/blog/2013-07/pagination-done-the-postgresql-way).

Unfortunately, Kaminari does not have a solution for this, at this moment.

## The Solution

It turns out solving this problem is not that difficult. It can be done in less than 100 lines of code [gist](https://gist.github.com/goromlagche/d2075bc8ad8fedd3a9930cd9f3c38ae4).

### Ordering

First, let us talk about ordering. Normally, a table's primary key is set to field `id` which is a `bigint` column, set to `autoincrement`. With an autoincrement primary key, ordering can be done on the field itself.

``` sql
select * from records
where id < cursor_id
order by id desc
limit 10;
```

But this will not work if you are using a truly unique `UUID` field (`uuid_generate_v4`) as your primary key. I am not going to talk about the benefits of using UUID in this blog, there are well-established points on both the pros and cons.

With UUID, we will need a different field to do the ordering. I would say using a timestamp field like `created_at` is a good choice.

``` sql
select * from records
where created_at < cursor_created_at
order by created_at desc
limit 10;
```

Well, using a timestamp field is still not a full-proof solution.

Postgres timestamp fields allow up to 6 fraction digits in seconds field. This roughly means the timestamp is accurate up to microseconds. If you have multiple records created at the same time, up to the same microsecond, this becomes a problem. Although the chances of this happening are pretty slim, still it can happen.

To fully solve this problem, I believe we can use a secondary `id` field, we can set it to `autoincrement`/create a `sequence`. And use that field for ordering. We can only use the integer field for an autoincrement sequence. Even if we set it to `bigint` there is a possibility, we might run out of numbers to allocate. In that case, we can create `id_v1`, `id_v2` and so on..

Ok, enough of HLD system design talk, let us get back to the problem at hand. For this article, I will just use `created_at` for ordering.

### API Interface

First, we will define our design/interface. We will be building this for a backend API. And we will need to make our API dead-simple so that integrating with a UI is trivial.

Through our API we will send a JSON block,

``` json
{
  "records": [
    {record_1},
    {record_2},
  ],
  "next_url": "/api/records?cursor_created_at={timestamp}&&direction=next",
  "prev_url": "/api/records?cursor_created_at={timestamp}&&direction=prev"
}
```

When using cursor-based pagination, we will need primarily 2 sets of values, which should be passed on as URL parameters,
1. **direction** (This can have only two values `prev` and `next`.)
2. **cursor_created_at** (Which will be used for ordering.)

### Seek (limit + 1)

Once we receive an api request, we can expect it to have URL params `direction` and `cursor_created_at`.

For the first request though, to grab the first 10 records, we won't need the URL params. First request `/api/records` will result in a sql query,

``` sql
select * from records
order by created_at desc
limit 11
```

Notice I am fetching 1 extra row from the db. This is useful to create the `next/prev url`. If the sql query returns (limit + 1) records, that means we have more records available.

### Going Forward and Backward

With sql query `offset`, going back is easy. When using cursor-based pagination, it is slightly more complicated.
Let us see the code first, then we can dig deep

``` ruby
def paginated_records
  paginated_records = records
                        .where(pagination_window)
                        .order(created_at: order_direction)
                        .limit(limit + 1)
                        .to_a

  seek = (paginated_records.size > limit)
  paginated_records.pop if seek == true
  paginated_records.reverse! if params[:direction] == "prev"
  paginated_records
end

def order_direction
  return :asc if params[:direction] == 'prev'

  :desc
end

def pagination_window
  return if params[:cursor_created_at].blank?

  if params[:direction] == "prev"
    records.arel_table[:created_at].gt(params[:cursor_created_at])
  else
    records.arel_table[:created_at].lt(params[:cursor_created_at])
  end
end
```

Let us say we have 6 records. Which were created between 1pm to 6pm, each an hour apart.

``` ruby
[1, 2, 3, 4, 5, 6]
```

We will set the page limit to 2. And we want to show the latest created record at the top of the page.
First page should be `[6, 5]`, if we want to go to `next` page, we should see `[4, 3]`

Once we get a request next `direction=next&&cursor_created_at=5`, the request will get converted to a sql query, like

``` sql
select * from records
where created_at < 5
order by created_at desc
limit 2
```

And it should return records

``` ruby
[4, 3]
```

We will click `next` page again, we should reach `[2, 1]`, from here if we want to go to prev page, we should see `[4, 3]`.

We get a request `direction=prev&&cursor_created_at=2`, notice we will need to modify the order.

``` sql
select * from records
where created_at > 2
order by created_at asc
limit 2
```

Will return

``` ruby
[3, 4]
```

When going `prev` we will need to reverse the fetched records one more time. To get the desired result. Hence the line

``` ruby
paginated_records.reverse! if params[:direction] == "prev"
```

### Pagination URLs

We will need to provide `next_url` and `prev_url` for the frontend to consume.

Fetching `limit + 1` is particularly helpful, when creating prev/next urls.

``` ruby
def pagination_url(direction:)
  return "" if seek == false && direction == params[:direction]
  return "" if params[:cursor_created_at].blank? && direction == "prev" # first page

  cursor = direction == "prev" ?  paginated_records.first : paginated_records.last

  uri = URI(url)
  params = Hash[URI.decode_www_form(uri.query || "")]
             .merge("cursor_created_at" => cursor.created_at.iso8601(6), # PG default is 6
                    "direction" => direction)
  uri.query = URI.encode_www_form(params)
  uri
end
```


All right, thats all I hade to share.

The full code is available [here](https://gist.github.com/goromlagche/d2075bc8ad8fedd3a9930cd9f3c38ae4#file-pagination-rb).

All you will need to do is drop the file in `app/lib/pagination.rb`. And you should be able to include the module in a controller.
I will also try to create a small gem around this.

Until next time! :heart:
