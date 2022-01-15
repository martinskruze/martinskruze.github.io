**TL;DR** Arel behaviour has not changed in examined examples. This black box testing approach has given me confidence that everything that has worked in Rails 6 will work in Rails 7.

# Does my old Arel queries work in Rails 7
I will admit a little secret - I like using _Arel_ (A Relational Algebra) as a Rails developer. Currently _Arel_ has reached it's 10th version and is a member of Rails codebase.
There have been a lot of rumors floating around about breaking changes for this _private_ Rails API. How true are those rumors and will our favorite _Arel_ requests still work in future - that is a question that I wanted to find some answers to.

Small disclaimer: do not use _Arel_ if you are not prepared to follow development of it and solve issues when upgrading Rails. It is a powerful tool, but as one web swinging superhero's uncle once said: _with great power comes great responsibilities_.

The method for madness below - just two applications connected to the same database and some queries inspired by my past applications. Although dates in examples are 2021-11, this still holds true to newly released rails 7.0.1 (as of January 2022).

We will use simple data structure: we have table users, which has id and username, articles which has id, author_id (the same user), subject and body, and comments with id, content, article_id, parent_id (other comments if nested) and commenter_id (some record from user) + created_at and updated_at timestamps for each of the tables. Please understand - the data structure is just for illustration and not for some serious work. Please do not judge me on this.

![Database structure](/files/does_arel_work_in_rails_7/database_structure.png)

All the tables have their own appropriate models: Application::User, Application::Article and Application::Comment (I added Application namespace in order to make them in different, mountable folder).
I added some data, you can see in the seed file. And there are tests covering it.

## Where I can play with code

The github repo [dis-button/arel_for_rails](https://github.com/dis-button/arel_for_rails) contains all the code used. Just docker-compose up and create, migrate and seed db and you are set to go.
Only weird thing - the common application code (models, service objects and specs) is in the “/general/...” folder, which contains parts to be mounted as volumes in docker (it might sound daunting, but basically - general folder - code - rails application folders - just to execute the code).

All of the code is divided into service objects. For example, the final version of Simple Requests Finding user is in general/app/services/simple_requests/finding_user.rb.

If you want to use code mentioned in this or parts of it - go for it, if you discuss it somewhere some mention would be appreciated!

For everything there are specs, if you want to just make changes and run - more info in README of that git repo.

## 1. SIMPLE REQUESTS
### Finding user
The simplest request that you can ask - let's find a user with the username of “martins.kruze”. (I know that you can do the thing with rails ActiveRecord query Application::User.find_by(username: “martins.kruze”), but where is the fun in that?
#### Arel query
```ruby
Application::User.find_by(Application::User.arel_table[:username].eq('martins.kruze'))
```
#### Rails 6 generated SQL
```sql
SELECT "users".*
FROM "users"
WHERE "users"."username" = 'martins.kruze'
LIMIT 1
```
#### Rails 7 generated SQL
```sql
SELECT "users".*
FROM "users"
WHERE "users"."username" = 'martins.kruze'
LIMIT 1
```
![good news](https://c.tenor.com/YLIPXr2qJFsAAAAC/futurama-professor.gif)

**IT IS THE SAME!**

### Finding articles
A bit more complex and less trivial approach to arel is using different comparison operators that usual suspects of rails ActiveRecod ones '=' or 'in'. Let's imagine that our app has search functionality, and we want to find all articles which were created yesterday or are older and have the word 'arel' in the name.
#### Arel query
```ruby
Application::Article
  .where(Application::Article.arel_table[:subject].matches('%arel%'))
  .where(Application::Article.arel_table[:created_at].lteq(Date.current - 1.day))
```
#### Rails 6 generated SQL
```sql
SELECT "articles".*
FROM "articles"
WHERE "articles"."subject" ILIKE '%arel%'
  AND "articles"."created_at" <= '2021-11-27
```
#### Rails 7 generated SQL
```sql
SELECT "articles".*
FROM "articles"
WHERE "articles"."subject" ILIKE '%arel%'
  AND "articles"."created_at" <= '2021-11-27
```
Sweet juices of pinecones! Again - the same thing - it works!

## 2. FUNCTIONS AND ATTRIBUTES
### Ordering Articles
Lets order something, Ordering articles by their body length in descending order seems like a nice challenge. It would make use of some postges function (length()). For displaying (not necessary for ordering itself, we can add an extra attribute to returned collections articles 'body_length’ which would be the same that we use for ordering.
#### Arel query
```ruby
Application::Article.
  select(
    Arel.star,
    Arel::Nodes::NamedFunction.new('length', [Application::Article.arel_table[:body]]).as('body_length')
  ).
  order(Arel::Nodes::NamedFunction.new('length', [Application::Article.arel_table[:body]]).desc)
```

#### Rails 6 generated SQL
```sql
SELECT *, length("articles"."body") AS body_length
FROM "articles"
ORDER BY length("articles"."body") DESC
```

#### Rails 6 ordered results (just subjects)
```ruby
FunctionsAndAttributes::OrderingArticles.call.pluck(:subject)
# (0.8ms)  SELECT "articles"."subject" FROM "articles" ORDER BY length("articles"."body") DESC
# => ["If - (1910)", "Charge (1854)", "The City of Sleep (1895)", "Pills - do not take them!", "Arel is great"]
```

#### Rails 6 is the returned attribute body_length accessible?
It is always been accessible
```ruby
a = FunctionsAndAttributes::OrderingArticles.call.first.body_length
#=> 1523
```
#### Rails 7
It continues to play nice - everything is the same

## 3. OR statements
### Articles by title or creation time
Lets find Articles where the subject has the name arel in it or is created more than 24 hours ago (DateTime.current is '2021-11-28 16:50:38.302849').
#### Arel query
```ruby
Application::Article.where(
  Application::Article.arel_table[:subject].matches('%arel%').
    or(Application::Article.arel_table[:created_at].lteq(DateTime.current - 1.day))
)
```
#### Rails 6 generated SQL
```sql
SELECT "articles".*
FROM "articles"
WHERE (
  "articles"."subject" ILIKE '%arel%'
  OR "articles"."created_at" <= '2021-11-27 16:50:38.302849'
)
```
#### Rails 7
IT WORKS AS EXPECTED!
This is very promising, let's continue for something more complex and convoluted.

## 4. UNION
This covers subqueries also. Side Note - **you should make use of [ActiveRecordExtended](https://github.com/GeorgeKaraszi/ActiveRecordExtended) gem**, if you require union, CTE or something more complex, as your apps functionality) or just do some plain sql requests. These examples are more of an engineering exercise than a real use case, although… …you can make some really perverted APIs… **Just imagine the possibilities…**

![hahahaha](https://c.tenor.com/Xdz61gZI5eYAAAAC/super-science-friends-laughing.gif)

### Biggest Provocateur
This is a bit of an abstract use case (sometimes clients can be really weird with their requests - we are not here to judge, but to encourage). Let's imagine that we want to find out which of our users brings the most interaction. By creating commentable articles (most commented) or by commenting himself. 

In this case - let's imagine that we want to get some comments for users' articles or comments that the user created by himself (in any article). This functionality can be realised in a lot of different ways - but let's try to use union (union not union all, because we want distinct comments).

Warning (a bit of offtopic)! In _real life_, if you want to use something similar, use Coment.arel_table as an alias for some subquery only if you really return only comments, otherwise it is a bit dubious at best. Rather some other alias or maybe select just ids and query Comment(id: subquery.arel).

We are using Union node here, but if you need in rails 6 (and 7) there is UnionAll node too.

#### Arel query:
```ruby
users_comments = Application::Comment.
  joins(:commenter).
  where(Application::User.arel_table[:username].eq('martins.kruze')).
  arel

users_articles_comments = Application::Comment.
  joins(article: :author).
  where(Application::User.arel_table[:username].eq('martins.kruze')).
  arel

unionized_results = Arel::Nodes::As.new(
  Arel::Nodes::Union.new(users_comments, users_articles_comments),
  Application::Comment.arel_table
)

Application::Comment.from(unionized_results)
```
#### Rails 6 generated SQL
```sql
SELECT "comments".*
FROM (
  (
    SELECT "comments".*
    FROM "comments"
    INNER JOIN "users"
      ON "users"."id" = "comments"."commenter_id"
    WHERE "users"."username" = 'martins.kruze'
  ) UNION (
    SELECT "comments".*
    FROM "comments"
    INNER JOIN "articles"
      ON "articles"."id" = "comments"."article_id"
    INNER JOIN "users"
      ON "users"."id" = "articles"."author_id"
    WHERE "users"."username" = 'martins.kruze'
  )
) AS "comments"
```
#### Rails 7
Drumroll… …it works! Good! So I have a couple more complex ideas… …have you heard about CTE?

## 5. Common Table Expressions
Common table expressions ('with as ()' sql statements) are useful tools in every complex SQL statement. These can get complicated, even with pg recursion, so I have two examples.  Side Note - **you should make use of [ActiveRecordExtended](https://github.com/GeorgeKaraszi/ActiveRecordExtended) gem**, if you require union, CTE or something more complex, as your apps functionality).

These examples are more of an engineering exercise than a real use case, but if there is a will, there is a way, so you do you.
### Order articles
In this example, let's try to order articles by longest comment. This is not the most efficient way to order articles by longest comment, but it is just to exemplify CTE usage. This will show usage of window function and custom joining on newly created attributes. All our found articles must have comments for simplicity's sake. We want to display not only articles, but also how long is the longest comment.
#### Rails 6 arel query with rails 6 sql
```ruby
# lets define new arel table
longest_comments_cte = Arel::Table.new(:longest_comments)

# lets define function to calculate comments length
comment_length_function = Arel::Nodes::NamedFunction.new('length', [Application::Comment.arel_table[:content]])
# comment_length_function.to_sql is:
#   length("comments"."content")

# lets define new sql window function that can be used for getting correct row number
longest_comment_window_function = Arel::Nodes::Window.new.
  partition(Application::Comment.arel_table[:article_id]).
  order(Arel::Nodes::NamedFunction.new('length', [Application::Comment.arel_table[:content]]).desc)
# longest_comment_window_function.to_sql is:
#   (PARTITION BY "comments"."article_id" ORDER BY length("comments"."content") DESC)

# lets define row number function over our window function
longest_comment_rank_function = Arel::Nodes::NamedFunction.
  new('row_number', []).
  over(longest_comment_window_function)
# longest_comment_rank_function.to_sql is:
#   row_number() OVER (PARTITION BY "comments"."article_id" ORDER BY length("comments"."content") DESC)

# lets define the CTE content
longest_comments_select = Application::Comment.arel_table.project(
  Application::Comment.arel_table[:article_id],
  comment_length_function.as('comment_length'),
  longest_comment_rank_function.eq(1).as('is_longest_comment')
)
# longest_comments_select.to_sql is:
#   SELECT
#     "comments"."article_id",
#     length("comments"."content") AS comment_length,
#     row_number() OVER (
#       PARTITION BY "comments"."article_id" ORDER BY length("comments"."content") DESC
#     ) = 1 AS is_longest_comment
#   FROM "comments"

# lets define the CTE itself
composed_longest_comments_cte = Arel::Nodes::As.new(
  longest_comments_cte,
  longest_comments_select
)
# longest_comments_select.to_sql is:
# "longest_comments" AS (
#   SELECT
#     "comments"."article_id",
#     length("comments"."content") AS comment_length,
#     row_number() OVER (
#       PARTITION BY "comments"."article_id" ORDER BY length("comments"."content") DESC
#     ) = 1 AS is_longest_comment
#   FROM "comments"
# )

# define our joining conditions
join_conditions = Application::Article.arel_table[:id].
  eq(longest_comments_cte[:article_id]).
  and(longest_comments_cte[:is_longest_comment])
# join_conditions.to_sql is:
# "articles"."id" = "longest_comments"."article_id" AND "longest_comments"."is_longest_comment"

# lets define the way articles will be joined to CTE
sql_statement = Application::Article.arel_table.
  project(Application::Article.arel_table[Arel.star], longest_comments_cte[:comment_length]).
  join(longest_comments_cte).
  on(join_conditions).
  order(longest_comments_cte[:comment_length].desc, Application::Article.arel_table[:created_at].desc).
  with(composed_longest_comments_cte)
# sql_statement.to_sql is:
# WITH "longest_comments" AS (
#   SELECT
#     "comments"."article_id",
#     length("comments"."content") AS comment_length,
#     row_number() OVER (
#       PARTITION BY "comments"."article_id"
#       ORDER BY
#         length("comments"."content") DESC
#     ) = 1 AS is_longest_comment
#   FROM
#     "comments"
# )
# SELECT
#   "articles".*,
#   "longest_comments"."comment_length"
# FROM "articles"
# INNER JOIN "longest_comments"
#   ON "articles"."id" = "longest_comments"."article_id"
#   AND "longest_comments"."is_longest_comment"
# ORDER BY
#   "longest_comments"."comment_length" DESC,
#   "articles"."created_at" DESC

Application::Article.find_by_sql(sql_statement)
# this finds all articles with their longest comment
# pp Application::Article.find_by_sql(sql_statement).map { |a| [a.subject, a.comment_length].join(": ") }:
# ["Pills - do not take them!: 43",
#   "Arel is great: 26",
#   "Charge (1854): 20",
#   "The City of Sleep (1895): 20",
#   "If - (1910): 20"]
```
#### Rails 7
By the Merlin's beard… IT WORKS… Ok, I have a last idea - lets mix multiple CTEs and add postgres recursion… Maybe that will kill it.

### Article with longest comment chain
To have little sql recursion fun, we added comment_id to the comments table. Let's imagine that we need to have the articles and have an “engagement score” (article_score in final sql), which is calculated by multiplying the comment thread count by the maximum comment thread size of the longest thread for this article. Articles need to be ordered by this.

There are multiple ways of doing this, but let's challenge ourselves (and _Arel_) with some fun with postgre recursion. This is not the perfect and most speedy solution, I just wanted to explore the generation of query with multiple CTE statements and recursion.
#### Rails 6 arel query with rails 6 sql
```ruby
# lets define new arel tables
# comment_chain - this will recursively help us determine the longest comment chain
comment_chain_table = Arel::Table.new(:comment_chain)
# article_max_chain_statistics - this will get max from comment_chain collection
article_max_chain_statistics_table = Arel::Table.new(:article_max_chain_statistics)
# article_thread_count - this will count all of the zero level comments - thread starts
article_thread_count_table = Arel::Table.new(:article_thread_count)

comment_chain_base_select = Application::Comment.arel_table.project(
  Application::Comment.arel_table[:id].as('super_parent_id'),
  Application::Comment.arel_table[:id].as('id'),
  Application::Comment.arel_table[:article_id].as('article_id'),
  Arel::Nodes::SqlLiteral.new('1').as('thread_length')
).where(Application::Comment.arel_table[:parent_id].eq(nil))
# puts comment_chain_base_select.to_sql
# SELECT
#   "comments"."id" AS super_parent_id,
#   "comments"."id" AS id,
#   "comments"."article_id" AS article_id,
#   1 AS thread_length
# FROM "comments"
# WHERE "comments"."parent_id" IS NOT NULL

comment_chain_recursive_select = Application::Comment.arel_table.project(
  comment_chain_table[:super_parent_id].as('super_parent_id'),
  Application::Comment.arel_table[:id].as('id'),
  Application::Comment.arel_table[:article_id].as('article_id'),
  Arel::Nodes::Addition.new(
    comment_chain_table[:thread_length],
    Arel::Nodes::SqlLiteral.new('1')
  ).as('thread_length')
).join(comment_chain_table).on(
  comment_chain_table[:id].eq(Application::Comment.arel_table[:parent_id])
)
# puts comment_chain_recursive_select.to_sql
# SELECT
#   "comment_chain"."super_parent_id" AS super_parent_id,
#   "comments"."id" AS id,
#   "comments"."article_id" AS article_id,
#   "comment_chain"."super_parent_id" + 1 AS thread_length
# FROM "comments"
# INNER JOIN "comment_chain"
#   ON "comment_chain"."id" = "comments"."parent_id"

comment_chain_cte = Arel::Nodes::As.new(
  comment_chain_table,
  Arel::Nodes::UnionAll.new(comment_chain_base_select, comment_chain_recursive_select)
)
# puts comment_chain_cte.to_sql
# "comment_chain" AS (
#   (
#     SELECT
#       "comments"."id" AS super_parent_id,
#       "comments"."id" AS id,
#       "comments"."article_id" AS article_id,
#       1 AS thread_length
#     FROM "comments"
#     WHERE "comments"."parent_id" IS NULL
#   ) UNION ALL (
#     SELECT
#       "comment_chain"."super_parent_id" AS super_parent_id,
#       "comments"."id" AS id,
#       "comments"."article_id" AS article_id,
#       "comment_chain"."super_parent_id" + 1 AS thread_length
#     FROM "comments"
#     INNER JOIN "comment_chain"
#       ON "comment_chain"."id" = "comments"."parent_id"
#   )
# )

# EXTRA NOTE:
# If I would like to debug this cte - what does it return, i can get to data like this:
# ActiveRecord::Base.connection.execute(
#   comment_chain_table.project(comment_chain_table[Arel.star]).
#   with(:recursive, comment_chain_cte).to_sql
# ).as_json

article_max_chain_statistics_select = comment_chain_table.project(
  comment_chain_table[:thread_length].maximum.as('max_thread_length'),
  comment_chain_table[:article_id].as('article_id')
).group(comment_chain_table[:article_id])
# article_max_chain_statistics_select.to_sql
# SELECT
#   MAX("comment_chain"."thread_length") AS max_thread_length,
#   "comment_chain"."article_id" AS article_id
# FROM "comment_chain"
# GROUP BY "comment_chain"."article_id"

article_max_chain_statistics_cte = Arel::Nodes::As.new(
  article_max_chain_statistics_table,
  article_max_chain_statistics_select
)
# puts article_max_chain_statistics_cte.to_sql
# "article_max_chain_statistics" AS (
#   SELECT
#     MAX("comment_chain"."thread_length") AS max_thread_length,
#     "comment_chain"."article_id" AS article_id
#   FROM "comment_chain"
#   GROUP BY "comment_chain"."article_id"
# )

article_thread_count_select = Application::Comment.arel_table.project(
  Arel::Nodes::SqlLiteral.new('1').count.as('thread_count'),
  Application::Comment.arel_table[:article_id].as('article_id')
).where(
  Application::Comment.arel_table[:parent_id].eq(nil)
).group(Application::Comment.arel_table[:article_id])
# puts article_thread_count_select.to_sql
# SELECT
#   COUNT(1) AS thread_count,
#   "comments"."article_id" AS article_id
# FROM "comments"
# WHERE "comments"."parent_id" IS NULL
# GROUP BY "comments"."article_id"

article_thread_count_cte = Arel::Nodes::As.new(
  article_thread_count_table,
  article_thread_count_select
)
# puts article_thread_count_cte.to_sql
# "article_thread_count" AS (
#   SELECT
#     COUNT(1) AS thread_count,
#     "comments"."article_id" AS article_id
#   FROM "comments"
#   WHERE "comments"."parent_id" IS NULL
#   GROUP BY "comments"."article_id"
# )

sql_statement = Application::Article.arel_table.
  project(
    Application::Article.arel_table[Arel.star],
    Arel::Nodes::Multiplication.new(
      article_max_chain_statistics_table[:max_thread_length],
      article_thread_count_table[:thread_count]
    ).as('article_score')
  ).
  join(article_max_chain_statistics_table).on(
    article_max_chain_statistics_table[:article_id].eq(Application::Article.arel_table[:id])
  ).
  join(article_thread_count_table).on(
    article_thread_count_table[:article_id].eq(Application::Article.arel_table[:id])
  ).
  with(
    :recursive,
    comment_chain_cte,
    article_max_chain_statistics_cte,
    article_thread_count_cte
  ).
  order(
    Arel::Nodes::Multiplication.new(
      article_max_chain_statistics_table[:max_thread_length],
      article_thread_count_table[:thread_count]
    ).desc
  )
# puts sql_statement.to_sql
# WITH RECURSIVE "comment_chain" AS (
#   (
#     SELECT
#       "comments"."id" AS super_parent_id,
#       "comments"."id" AS id,
#       "comments"."article_id" AS article_id,
#       1 AS thread_length
#     FROM
#       "comments"
#     WHERE
#       "comments"."parent_id" IS NOT NULL
#   )
#   UNION ALL (
#     SELECT
#       "comment_chain"."super_parent_id" AS super_parent_id,
#       "comments"."id" AS id,
#       "comments"."article_id" AS article_id,
#       "comment_chain"."super_parent_id" + 1 AS thread_length
#     FROM
#       "comments"
#       INNER JOIN "comment_chain" ON "comment_chain"."id" = "comments"."parent_id"
#   )
# ),
# "article_max_chain_statistics" AS (
#   SELECT
#     MAX("comment_chain"."thread_length") AS max_thread_length,
#     "comment_chain"."article_id" AS article_id
#   FROM
#     "comment_chain"
#   GROUP BY
#     "comment_chain"."article_id"
# ),
# "article_thread_count" AS (
#   SELECT
#     COUNT(1) AS thread_count,
#     "comments"."article_id" AS article_id
#   FROM
#     "comments"
#   WHERE
#     "comments"."parent_id" IS NULL
#   GROUP BY
#     "comments"."article_id"
# )
# SELECT
#   "articles".*,
#   "article_max_chain_statistics"."max_thread_length" * "article_thread_count"."thread_count" AS article_score
# FROM
#   "articles"
#   INNER JOIN "article_max_chain_statistics" ON "article_max_chain_statistics"."article_id" = "articles"."id"
#   INNER JOIN "article_thread_count" ON "article_thread_count"."article_id" = "articles"."id"
# ORDER BY "article_max_chain_statistics"."max_thread_length" * "article_thread_count"."thread_count" DESC

Application::Article.find_by_sql(sql_statement)
```
#### Rails 7
OK… Nothing will change then, maybe next time.

## CONCLUSION
As far as I can see - Arel in Rails 7 works with the same requests as Rails 6, but I have a dockerised framework for comparison purposes… What should I try next?
