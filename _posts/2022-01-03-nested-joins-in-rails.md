---
layout: posts
title:  "Nested joins in rails"
excerpt: How we can do nested joins using rails
date:   2022-01-03 03:00:47 +0530
layout: single
tags: [rails, active_record, database]
---
Recently while contributing to [RubyForGood](https://github.com/rubyforgood/inkind-admin/), I came across a problem, requiring nested joins. I had not faced a similar situation before. And to my surprise, `ActiveRecord` seems to be able to handle this quite easily.

## Premise

[InKind](https://github.com/rubyforgood/inkind-admin/) is an application built to serve [Community Education Partnerships](https://www.cep.ngo/). This application provides an admin interface to manage volunteers and students.

A `Student` is assigned an active `Staff`(User) through `StudentStaffAssignment`.

``` ruby
class Student
  has_many :student_staff_assignments
  has_one :active_student_staff_assignment,
          -> { where("student_staff_assignments.end_date > ?", Date.current) },
          class_name: "StudentStaffAssignment"
  has_one :staff, through: :active_student_staff_assignment
end
```

``` ruby
class StudentStaffAssignment < ApplicationRecord
  belongs_to :student
  belongs_to :staff, class_name: "User",
                     foreign_key: :staff_id,
                     inverse_of: :student_staff_assignments
end
```

A `Student` can respond to a `Survey`.

``` ruby
class SurveyResponse < ApplicationRecord
  belongs_to :student
  belongs_to :survey
end
```

There can be `SupportTickets` created, based on `SurveyResponse` provided by the student.

``` ruby
class SupportTicket < ApplicationRecord
  belongs_to :survey_response,
             foreign_key: :survey_response_id,
             optional: true
end
```

## Problem and "the solution"

If we need the name of active `staff` assigned to the `student` for a `support_ticket` we can get it pretty easily,

``` ruby
ticket = SupportTicket.find(params[:id])
ticket.survey_response&.student&.active_student_staff_assignment&.staff&.name
```

It becomes tricky, when we need to sort all the tickets in the active `staff`'s name.

First, we will need to get all the data, which will need nested joins.

``` ruby
SupportTicket
  .left_outer_joins(
  :requestor,
  survey_response: {
    student: {
      active_student_staff_assignment: :staff
    }
  })
```

If you notice, we are not just doing nested joins. We are also using `scopes` and table alias. And `ActiveRecord` just works<sup><b>TM</b></sup>

It is pretty neat!

Next up, for sorting. We want to sort on `staff.last_name`.
We can print the SQL query using `.to_sql` and study the table structures.

``` sql
SELECT "support_tickets".* FROM "support_tickets"
LEFT OUTER JOIN "users" ON "users"."id" = "support_tickets"."requestor_id"
LEFT OUTER JOIN "survey_responses" ON "survey_responses"."id" = "support_tickets"."survey_response_id"
LEFT OUTER JOIN "students" ON "students"."id" = "survey_responses"."student_id"
LEFT OUTER JOIN "student_staff_assignments" ON "student_staff_assignments"."student_id" = "students"."id"
                AND (student_staff_assignments.end_date > '2022-01-03')
LEFT OUTER JOIN "users" "staffs_student_staff_assignments" ON "staffs_student_staff_assignments"."id" = "student_staff_assignments"."staff_id"
```

From the above, we can see that the `staffs_student_staff_assignments` table has the active staff information.
And we can sort using,

``` ruby
SupportTicket
  .left_outer_joins(
  :requestor,
  survey_response: {
    student: {
      active_student_staff_assignment: :staff
    }
  })
  .order('staffs_student_staff_assignments.last_name' => :desc)
```

Which generates the query,

``` ruby
SELECT "support_tickets".* FROM "support_tickets"
LEFT OUTER JOIN "users" ON "users"."id" = "support_tickets"."requestor_id"
LEFT OUTER JOIN "survey_responses" ON "survey_responses"."id" = "support_tickets"."survey_response_id"
LEFT OUTER JOIN "students" ON "students"."id" = "survey_responses"."student_id"
LEFT OUTER JOIN "student_staff_assignments" ON "student_staff_assignments"."student_id" = "students"."id"
                AND (student_staff_assignments.end_date > '2022-01-03')
LEFT OUTER JOIN "users" "staffs_student_staff_assignments" ON "staffs_student_staff_assignments"."id" = "student_staff_assignments"."staff_id"
ORDER BY "staffs_student_staff_assignments"."last_name" DESC
```

Alright, that is all I had to share today.

Until next week! :heart:
