---
layout: posts
title:  "Rails schema collation and mysql 5.7"
excerpt: Rails mysql adapter sometimes fails to get collation value for mysql 5.7
date:   2022-03-09 17:20:47 +0530
layout: single
tags: [rails, mysql, active-record]
---
If you have a collation value set to `utf8mb4_general_ci` and use MySQL 5.7. On running `rails db:migrate` in local env, you will notice, `create_table` lines changes.

**Before** ⇒ `create_table "test", charset: "utf8mb4", collation: "utf8mb4_general_ci"`

**After** ⇒ `create_table "test", charset: "utf8mb4"`

The collation information is gone. And this issue is limited to collation value `utf8mb4_general_ci` This issue does not show up when collation is `utf8mb4_unicode_ci` or `utf8mb4_bin` or any other collation.

## The Cause
In rails MySQL adapter `SHOW CREATE TABLE` statement is used to figure out the [collation of a table](https://github.com/rails/rails/blob/6-1-stable/activerecord/lib/active_record/connection_adapters/abstract_mysql_adapter.rb#L802-L804).

And a regex is used to extract the [collation value](https://github.com/rails/rails/blob/6-1-stable/activerecord/lib/active_record/connection_adapters/abstract_mysql_adapter.rb#L465-L469).

You will see, `mysql:5.7.34` or any version before `mysql:8` does not show the collation value if it is set to `utf8mb4_general_ci.`

![mysql_5_7_23_collation_issue](/assets/images/mysql_5_7_34_collation_2022-03-09-16:18:19.png )

For `mysql:5.7.34` the default collation is `utf8mb4_general_ci`.

![mysql_5_7_23_default_collation](/assets/images/mysql_5_7_34_default_collation_2022-03-09-17:31:28.png)

## Solutions

One solution that comes to mind is you can change the collation. The problem with this will be an `ALTER TABLE` command. It will take a lot of time/other issues might show up. Need to do it carefully, tools like `pt-online-schema-change` might help.

If you indeed change the collation, another question will be which collation to choose. As `utf8mb4_general_ci` and `utf8mb4_unicode_ci` both have the `sushi-beer problem`, `utf8mb4_bin` should be a good candidate.

Resources
1. [MySQL Bugs: #76553: Sushi-Beer issue of MySQL with utf8mb4](https://bugs.mysql.com/bug.php?id=76553)
2. [MySQL :: Sushi = Beer ?! An introduction of UTF8 support in MySQL 8.0](https://dev.mysql.com/blog-archive/sushi-beer-an-introduction-of-utf8-support-in-mysql-8-0/)

Another solution will be to upgrade MySQL to version 8. If you do upgrade to MySQL 8, consider using collation `utf8mb4_0900_ai_ci` which is the default in MySQL 8

You can also just update the schema.rb without the collation information(when collation is `utf8mb4_general_ci`) as that is the default collation in MySQL 5.7.



Until next week! :heart:
