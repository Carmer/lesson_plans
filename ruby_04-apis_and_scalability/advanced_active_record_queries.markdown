---
title: Advanced ActiveRecord Queries
length: 180
tags: ruby, rails, activerecord, sql
---

## Learning Goals

* See some techniques for handling less common, more complicated
  ActiveRecord queries
* Get more practice expressing complex relational logic via ARel

## Structure

* 10 - Warmup / Recap -- review previously covered material from Queries
  lesson; self-assessment quiz
* 15 - Queries Lesson finishing up -- Fetching less data
* 5 - break
* 15 - Queries Lesson finishing up -- Rethinking data storage
  (serialization)
* 10 - Queries exercises
* 5 - break
* 25 - Queries exercises
* 5 - break
* 25 - Additional topics
* 5 - break
* 25 - Homework review / Advanced Query Q&A

### Recap

Let's briefly revisit our [Query Performance Lesson](http://tutorials.jumpstartlab.com/topics/performance/queries.html) from a few weeks ago.

Here are the topics we covered:

* Using `#to_sql` on an ARel query to expose the exact sql it will
  generate
* Using indices to improve query performance on specific columns
* Using `#includes` on ARel queries to pre-fetch related data with a
  query
* Using counter caches to improve performance of frequently-counted data

Let's jog our brains on this material with a quick recap quiz.

#### Recap Quiz

Go through the questions in this quiz to see how much you remember from
the previous session (none of this is tracked or graded; it's just a tool to help jog your memory): https://turing-quiz.herokuapp.com/quizzes/query-perf-recap.

### Finishing Up Query Performance Topcis

We didn't get into the final 2 topics from our previous Query
Performance Lesson. Let's discuss them now:

* [Fetching Less Data](http://tutorials.jumpstartlab.com/topics/performance/queries.html#fetching-less-data)
* [Rethinking Data Storage](http://tutorials.jumpstartlab.com/topics/performance/queries.html#rethinking-data-storage)

### Exercises

Let's get some more hands-on experience with improving query performance
by working through the exercises in the tutorial: [http://tutorials.jumpstartlab.com/topics/performance/queries.html#exercises](http://tutorials.jumpstartlab.com/topics/performance/queries.html#exercises)


### Additional Topics

#### Recap: Joins vs. Includes

Recall that `#includes` is a handy technique for avoiding N+1 queries by
pre-fetching associated data.

Consider our previous example using comments and approvals:

```
a = Article.includes(comments: :approval).first
a.comments.select{|c| c.approved?}.count
```

Here we are actually pre-fetching data for 2 related models along with
our article. The Comment records (associated to our article by a foreign
key and a belongs_to association) and the Approval records (associated
to articles only via the intermediate comment records).

This allows us to avoid making additional queries later if we want to
display the approval or comment data itself (in a nested partial, for
example). But let's check out the queries ARel is performing here:

```
Article Load (0.1ms)  SELECT "articles".* FROM "articles" LIMIT 1
Comment Load (0.1ms)  SELECT "comments".* FROM "comments" WHERE "comments"."article_id" IN (8)
Approval Load (0.2ms)  SELECT "approvals".* FROM "approvals" WHERE "approvals"."comment_id" IN (6, 7, 8)
```

Notice that for each model, we are running a `SELECT....* FROM...` on
the corresponding table. This is great if we intend to actually use the
data (rendering it in a UI or some such), but if we aren't using the
data, it's a bit wasteful.

Let's look at an example where we might want to query against only a
specific portion of the data. Suppose we wanted to find all the comments
which have been approved. Our current technique of using `includes`
allows us to efficiently find the approval information for a specific
comment. But it doesn't help us much with querying against the combined
comment-approval data in bulk.

To do this, we might use `joins` to effectively combine the 2 tables and
then query against all of it at once. So, for example, to find only the
comments approved by user 1:

```
Comment.joins(:approval).where(approvals: {approved_by: 1}).count
```

Here, thanks to the joins, we are able to query against data from the
comments table and the approvals table at the same time. This ability to
perform queries across multiple tables in the DB is ultimately what
makes relational databases so powerful, and the `joins` method is our
main way for accessing this power through rails.

In Summary:

* Includes -- easier to use, less fine grained -- "grab everything just
  in case"
* Joins -- allows greater control but requires more specificity -- "let
  me avoid bloat by specifying exactly what I need"
* Includes intentionally uses multiple queries to fetch all the required
  data (and then caches this data in memory)
* Joins uses actual SQL joins to allow us to address multiple tables in
  a single query

#### Using references to automate creating assocations

As of Rails 4, The `ActiveRecord::Migration` table creation system now includes a
`references` method which automates creating foreign keys for
associations.

So in the past we have always created associations in a migration with
this approach:

```
class CreatePizzas < ActiveRecord::Migration
  def change
    create_table :pizzas do |t|
      t.string  :name
      t.integer :pizza_chef_id
      t.timestamps
    end
  end
end
```

Using the Rails 4 syntax, we could simply specify:

```
class CreatePizzas < ActiveRecord::Migration
  def change
    create_table :pizzas do |t|
      t.string  :name
      t.references :pizza_chefs
      t.timestamps
    end
  end
end
```

and Rails will name the column for us. This is especially useful when
generating a model, since Rails can also add the ActiveRecord
association methods as well:

```
rails g model Pizza pizza_chef:references
```

This will give us a migration including the reference column:

```
class CreatePizzas < ActiveRecord::Migration
  def change
    create_table :pizzas do |t|
      t.references :pizza_chef, index: true

      t.timestamps
    end
  end
end
```

And a `pizza.rb` model file with the appropriate `belongs_to`
association already added:

```
class Pizza < ActiveRecord::Base
  belongs_to :pizza_chef
end
```

Ultimately this is simply a time-saving technique. In some situations
you may even prefer the explicitness of adding them manually, but if
you're confident about the assocations you need to set up, using
references can save you a few seconds.

#### Scopes with Arguments

We've discussed scopes several times in the past, most often for
pre-configuring common queries to be run against specific column states
(find me all the orders which have the status "paid", all the articles
published on today's date, etc).

But scopes aren't limited to querying against static data values --
thanks to the fact that they're implemented using lambdas, we can define
scopes which accept arguments as well.

For example, suppose we wanted to allow users to find only Articles
created after a specific date. To do this, it would be handy if we had a
scope which was limited not just to Today's date, but to any variable
date we might pass in.

This can be done using a scope argument:

```
class Article < ActiveRecord::Base
  scope :published_after, ->(time) { where("created_at > ?", time) }
end
```

```
Article.published_after(10.months.ago).count
```

#### Homework Problem Recaps

Remember these lovely AR homework problems? Let's revisit them to
discuss some that we didn't get to before and to see if there are any
new questions that come up: https://gist.github.com/stevekinney/7bd5f77f87be12bd7cc6.

#### Notes For Next Time

This session was first added for the 2/15 - 3/15 module. During several
previous ActiveRecord sessions (https://github.com/turingschool/lesson_plans/blob/master/ruby_04-apis_and_scalability/query_performance.markdown, and homework review for https://gist.github.com/stevekinney/7bd5f77f87be12bd7cc6) we had run out of time with 1409 before working through all of the material.

This session served as a bit of mopping up to cover the remaining
material from those sessions, and to try to get students some more
practice writing more complicated AR queries in general.

Ultimately it would probably make sense to break some of these
monolithic AR sessions into multiple smaller sessions on specific topics
so that we are better able to cover all the material. But for the moment
this is how this particular lesson came to be and why it's a bit of a
hodgepodge.
