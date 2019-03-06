---
layout: post
title: Calculating Frequency Distribution in PostgreSQL
category: tutorials
tags:
- PostgreSQL
- Statistics
---

I've always loved statistics. Being a software engineer has allowed me a lot of opportunities
to solve complex statistical problems in SQL. With programming, we want to consider the current state of data which
means running calculations on demand. Recently, I had the opportunity to calculate
frequency distribution in SQL and wanted to share what I've learned with you.
<!--excerpt-->

## An example

Alright so before we just dive in, first we need an example. A good example...

While walking down a dirt path, a gentle breeze brushes against your neck. Suddenly,
out of nowhere a grizzled old bridgekeeper blocks your way. "Stop!" he exclaims. 

His breath reeks of elderberry and he smells faintly of hamster. As he stares
at you with milky white eyes he states the following:

"Who would cross the Bridge of Death must answer me these questions three, ere the other side he see."

To which you of course respond, "Ask me the questions, bridgekeeper. I'm not afraid.". So ask away
he does:

<blockquote>
  <ul>
    <li>What is your favorite color?</li>
    <li>What is your quest?</li>
    <li>What is the airspeed velocity of an unladen swallow?</li>
  </ul>
</blockquote>

As you stare at him puzzled, he (helpfully) gives you multiple choices for each question.
Each question now has a set of options: a, b, c, and d. Considering your options you answer
that your favorite color is indeed B) fuschia, your quest is A) not being hassled by creepy
bridgekeepers, and you answer for the airspeed velocity of an unladen swallow with A) An 
African or European swallow?

While certainly an unsettling experience, you're now really curious as to how other 
travelers before you answered the questions. So curious in fact that you're thinking 
about finding out what the frequency distribution for each question's choice was.

Ok, ok, ok. So this wasn't an actual situation I ran into at work but work with me here.

## Basic Frequency Distribution

<blockquote class="Info Info-right"><strong>What is Frequency Distribution?</strong><br />
  Frequency distribution is calculating the frequency (also known as rate or occurence) of 
  which question choice was selected 
</blockquote>

Think back on the Bridgekeeper's questions, there are a set number of options: a, b, c, and d
for each question and a set number of travelers whom answered. **Frequency
distribution is measuring the total times that each multiple choice option was selected.** You can think of each option (a, b, c, d) as a different category that we'd like
to group travelers into based on their answer.

Now that we're interested in analyzed previous traveler's responses we'll want
to know how well they answered. From looking at the frequency distribution, we
could also hypothesize if a question was too hard or too easy.

First let's write some SQL to gather traveler responses across all three
questions.

{% highlight sql %}
SELECT 
  traveler_answers.choice, 
  COUNT(traveler_answers.choice) as total_answers
FROM traveler_answers
GROUP BY traveler_answers.choice
ORDER BY traveler_answers.choice -- To ensure the order is a, b, c, d
{% endhighlight %}

The above query returns the choice (a, b, c, or d) along with a count of
how many times a traveler selected it. We group the above by `traveler_answers.choice` 
as this ensures that the aggregate function `COUNT()` returns a result for each type 
of choice. The query above nets us a basic frequency distribution
which you can see below:

| choice        | total_answers    |
|---------------|------------------|
| a             |   25            |
| b             |   50            |
| c             |   25            |
| d             |   75            |

Since this is frequency distribution at its most basic, it is pretty straight forward
to write the supporting SQL. However, it isn't really useful in its current form. What
does 25 total answers for choice A really tell us? Well, across the five questions... I mean
three questions, 25 travelers select choice A. Not real compelling stuff. 

"Well, lets get going! Come along, Patsy"

## Frequency Distribution for two categories

What might be more useful than frequency distribution across all questions? How
about frequency distribution based on a single question? Specifically let's calculate how many travelers
selected a **given choice** for a **given question**. You could also think about this
as comparing two categories within frequency distribution.

To accomplish this we follow along with our above example by adding a new `GROUP BY`
clause to ensure that we are grouping categories into traveler's answered choice
(`traveler_answers.choice`)
and the Bridgekeeper's question (`bridgekeeper_questions.question_text`).

{% highlight sql %}
SELECT 
  bridgekeeper_questions.question_text,
  traveler_answers.choice, 
  COUNT(traveler_answers.choice) as total_answers
FROM traveler_answers
INNER JOIN bridgekeeper_questions ON bridgekeeper_questions.id = traveler_answers.exam_question_id
GROUP BY 
  traveler_answers.choice, 
  bridgekeeper_questions.question_text
ORDER BY traveler_answers.choice
{% endhighlight %}

Since we're grouping the results by `traveler_answers.choice` and `bridgekeeper_questions.question_text`, we end
up with multiple results per question text (1 per choice / question pair). You can see
what the resulting query would return below.

| Question text | choice        | total_answers    |
|--------------------------------|---------------|------------------|
| "What is your favorite color?" | a             |   12            |
| "What is your favorite color?" | b             |   20            |
| "What is your favorite color?" | c             |   10            |
| "What is your favorite color?" | d             |   5            |
| "What is your quest?" | a             |   12            |
| "What is your quest?" | b             |   5            |
| "What is your quest?" | c             |   15            |
| "What is your quest?" | d             |   40            |
| "What is the airspeed velocity of an unladen swallow?" | a             |   1            |
| "What is the airspeed velocity of an unladen swallow?" | b             |   25            |
| "What is the airspeed velocity of an unladen swallow?" | c             |   5            |
| "What is the airspeed velocity of an unladen swallow?" | d             |   25            |

Now we have frequency distribution for each question. This is starting to look like useful data.

## Frequency Distribution with Descriptive Statistics

<blockquote class="Info Info-right clearfix"><strong>What is Descriptive Statistics?</strong><br />
  "A descriptive statistic (in the count noun sense) is a summary statistic that quantitatively describes or summarizes features of a collection of information"
<cite><a href="https://en.wikipedia.org/wiki/Descriptive_statistics">- Wikipedia</a></cite>
</blockquote>

At this point we've forgotten something. Who cares about a previous traveler's answer
unless we know if their selection was correct or not. "Who are
 you, who are so wise in the ways of science?". Totally normally if you're thinking that. 
 Well... probably not, but don't ruin my delusions.

Here's our new analysis goals:

1. What is the total number of correct answers per question?
2. What is the percentage of correct answers per question?

Now these won't make a lot of sense if for each row in the above table we return
total number and percentage of total correct. Really we want these in the format of:

`Question text | choice | total answers | total correct answers | percentage correct answers`

So you might say that the sh**rub**bery here is that we need to condense the multi-row format above
into a single line per question.

{% highlight sql %}
SELECT 
  inner_query.question_text, 
  SUM(inner_query.total_answers) as total_answers,
  SUM(inner_query.total_correct_answers) as total_correct_answers, 
  (SUM(inner_query.total_correct_answers) / SUM(inner_query.total_answers)) * 100 as percentage_correct_answer
FROM (
  SELECT 
    bridgekeeper_questions.question_text,
    traveler_answers.choice, 
    COUNT(traveler_answers.choice) as total_answers
    COUNT(traveler_answers.choice) FILTER (WHERE bridgekeeper_questions.correct_choice = traveler_answers.choice) as total_correct_answers
  FROM traveler_answers
  INNER JOIN bridgekeeper_questions ON bridgekeeper_questions.id = traveler_answers.exam_question_id
  GROUP BY 
    traveler_answers.choice, 
    bridgekeeper_questions.question_text
  ORDER BY traveler_answers.choice
) inner_query
GROUP BY inner_query.question_text
{% endhighlight %}

<blockquote class="Info Info-right"><strong>What's the Difference between sum and count?</strong><br /><br />
  <strong>COUNT</strong> - returns the <strong>total number</strong> of rows in the current group<br /><br />
  <strong>SUM</strong> - returns the <strong>total value</strong> from a column in the current group
</blockquote>

You'll notice that from the above query we used the syntax
`SELECT columns FROM (inner_query)` in order to select and aggregate already existing
aggregate values. Think of this like querying a query. This is necessary for us
to calculate an aggregate based on a group of multiple aggregates. It also allows
us to return a single row again per question.

To understand how we're back to returning a single row, let's look at what is happening behind the scenes by calculating the percentage
of correct answers. The formula for calculating percentage of correct answers is:

<blockquote>
  ( total_correct_answers / total_answers ) * 100
</blockquote>
 
First the inner query calculates the individual number of answers
per question. `COUNT(traveler_answers.choice)` returns:

| Question text | choice        | total_answers    |
|--------------------------------|---------------|------------------|
| "What is your favorite color?" | a             |   12            |
| "What is your favorite color?" | b             |   20            |
| "What is your favorite color?" | c             |   10            |
| "What is your favorite color?" | d             |   5            |

Additionally, we're counting correct answers by choice using a FILTER clause: 
`COUNT(traveler_answers.choice) FILTER (WHERE bridgekeeper_questions.correct_choice = traveler_answers.choice)`. The
results within the inner query here are a bit strange but will become useful once we aggregate its
results in the outer query.

| Question text | choice        | total_answers    | total_correct_answers |
|--------------------------------|---------------|------------------|------------------|
| "What is your favorite color?" | a             |   12            | 0 |
| "What is your favorite color?" | b             |   20            | 20 |
| "What is your favorite color?" | c             |   10            | 0 |
| "What is your favorite color?" | d             |   5            | 0 |

Think about the above as building out raw data that we want to perform calculations on in
the outer query. This is exactly what we'll do by leaning on the `SUM()` function 
with the following: `(SUM(inner_query.total_correct_answers) / SUM(inner_query.total_answers)) * 100`.

<blockquote class="Info Info-right"><strong>PostgreSQL FILTER clause</strong><br /><br />
  Something else you may have noticed is the `FILTER` syntax. This is an excellent way
  of extending an aggregate function with an additional `WHERE` clause. 
  <cite><a href="https://modern-sql.com/feature/filter">- Modern-sql.com</a></cite>
</blockquote>

We use `SUM()` instead of `COUNT()` because `COUNT()` would only give us how many rows were returned 
while we want to know what the running total is based on a column's value. 

Looking back we can 
see that we're right at the step for using our formula for calculating the
percentage.

Here's it written out in full:

{% highlight text %}
total_correct_answers: 0 + 20 + 0 + 0 = 20
total_answers: 12 + 20 + 10 + 5 = 47
(total_correct_answers / total_answers): (20 / 47) = 0.425532
(0.425532) * 100 = 42.5532%
{% endhighlight %}

Given our newest SQL changes above, here's the end result:

| Question text | total_correct_answers   | percentage_correct_answer    |
|---------------|-------------------------|---------------------------|
| "What is your favorite color?" | 20             |   42.5532            |
| "What is your quest?" | 40             |   55.5555            |
| "What is the airspeed velocity of an unladen swallow?" | 1             |   1.78571            |

Unfortunately, by running the aggregate for
total correct and percentage correct, we lost the frequency distribution for 
each question. It's a bit tricky to get this back but where there's a way... 

"The Black Knights always triumph!"

Ahem, excuse me. Sorry about that.

## Frequency Distribution as a single row

<blockquote class="Info Info-right"><strong>PostgreSQL JSONB methods</strong><br /><br />
  <strong>jsonb_agg</strong> - aggregates values as a JSON array<br /><br />
  <strong>jsonb_build_object</strong> - Builds a JSON object out of a variadic argument list. By convention, the argument list consists of alternating keys and values.
  <cite><a href="https://www.postgresql.org/docs/9.5/functions-json.html">- Postgresql.org</a></cite>
</blockquote>

Since we still want to return a single row per question, we need a way to format
the distribution into a single value. JSONB is a perfect solution for this problem
as it allows us to use key-value pairs to describe the value in a single column. 
We'll be using `jsonb_agg()` and `jsonb_build_object()` PostgreSQL methods to accomplish this. 

Just like we did above while calculating **total_correct_answers** and **percentage_correct_answer**, 
the aggregate of the inner query is utilized as an aggregate of the outer query.

{% highlight sql %}
SELECT 
  inner_query.question_text,
  SUM(inner_query.total_answers) as total_answers, 
  SUM(inner_query.total_correct_answers) as total_correct_answers, 
  (SUM(inner_query.total_correct_answers) / SUM(inner_query.total_answers)) * 100 as percentage_correct_answer,
  jsonb_agg(inner_query.frequency_distribution) as frequency_distribution_json
FROM (
  SELECT 
    bridgekeeper_questions.question_text,
    traveler_answers.choice, 
    COUNT(traveler_answers.choice) as total_answers,
    COUNT(traveler_answers.choice) FILTER (WHERE bridgekeeper_questions.correct_choice = traveler_answers.choice) as total_correct_answers,
    jsonb_build_object('choice', traveler_answers.choice, 'frequency', COUNT(traveler_answers.choice)) as frequency_distribution
  FROM traveler_answers
  INNER JOIN bridgekeeper_questions ON bridgekeeper_questions.id = traveler_answers.exam_question_id
  GROUP BY 
    traveler_answers.choice, 
    bridgekeeper_questions.question_text
  ORDER BY traveler_answers.choice
) inner_query
GROUP BY inner_query.question_text
{% endhighlight %}

"Oh,look. There's some lovely filth over here." 

The resulting dataset below is pretty ugly but gathers the proper key-value pairs
we need in order to show frequency distribution.

| Question text | total_correct_answers   | percentage_correct_answer    | frequency_distribution_json |
|---------------|-------------------------|---------------------------|-----------------------------|
| "What is your favorite color?" | 20             |   42.5532            | `[{ "choice": "a", "frequency": 12 }, { "choice": "b", "frequency": 20 }, { "choice": "c", "frequency": 10 }, { "choice": "d", "frequency": 5 }]` |
| "What is your quest?" | 40             |   55.5555            | `[{ "choice": "a", "frequency": 12 }, { "choice": "b", "frequency": 5 }, { "choice": "c", "frequency": 15 }, { "choice": "d", "frequency": 40 }]` |
| "What is the airspeed velocity of an unladen swallow?" | 1             |   1.78571            | `[{ "choice": "a", "frequency": 1 }, { "choice": "b", "frequency": 25 }, { "choice": "c", "frequency": 5 }, { "choice": "d", "frequency": 25 }]` |

With that being our final modification to the query, we now have a table that contains:

<blockquote class="Info Info-right"><strong>PostgreSQL alternative method</strong><br /><br />
  `width_bucket()` allows you to specify a column that has a minimum and maximum range and then split it into
  buckets (categories). <a href="https://www.postgresql.org/docs/9.1/functions-math.html">Check out the following documentation</a>
</blockquote>

1. The total correct answers per question
2. The percentage of correct answers per question
3. A frequency distribution per question

So what does the inner query look like with the new `jsonb_build_object()` method?

If we were to back track to the table that contained a result per question per 
choice and look at what the following SQL returns it would look like this:

{% highlight sql %}
  jsonb_build_object('choice', traveler_answers.choice, 'frequency', COUNT(traveler_answers.choice)) as frequency_distribution
{% endhighlight %}

| Question text | choice        | total_answers    | frequency_distribution |
|--------------------------------|---------------|------------------|------------------|
| "What is your favorite color?" | a             |   12            | `{ "choice": "a", "frequency": 12 }` |
| "What is your favorite color?" | b             |   20            | `{ "choice": "b", "frequency": 20 }` |
| "What is your favorite color?" | c             |   10            | `{ "choice": "c", "frequency": 10 }` |
| "What is your favorite color?" | d             |   5            | `{ "choice": "d", "frequency": 5 }` |
| "What is your quest?" | a             |   12            | `{ "choice": "a", "frequency": 12 }` |
| "What is your quest?" | b             |   5            | `{ "choice": "b", "frequency": 5 }` |
| "What is your quest?" | c             |   15            | `{ "choice": "c", "frequency": 15 }` |
| "What is your quest?" | d             |   40            | `{ "choice": "d", "frequency": 40 }` |
| "What is the airspeed velocity of an unladen swallow?" | a             |   1            | `{ "choice": "a", "frequency": 1 }` |
| "What is the airspeed velocity of an unladen swallow?" | b             |   25            | `{ "choice": "b", "frequency": 25 }` |
| "What is the airspeed velocity of an unladen swallow?" | c             |   5            | `{ "choice": "c", "frequency": 5 }` |
| "What is the airspeed velocity of an unladen swallow?" | d             |   25            | `{ "choice": "d", "frequency": 25 }` |

The magic... "I'm not a witch, I'm not a witch"... of this happens in the outer query with `jsonb_agg()` which
aggregates all the inner frequency distribution columns into an array
of jsonb key-value pairs like we see in the above final query. This allows
for both Descriptive Statistics on the entire dataset while providing Frequency Distribution
for a single question.

<blockquote>
  "What if you get a question wrong?"<br>
  "Then you are cast into the Gorge of Eternal Peril."
</blockquote>

![Monty Python Bridgekeeper](/img/2019/monty-python-bridge.gif)

Got a different SQL statistics trick? How about an improved approach to the above? 
Just love Monty Python? I'd love to hear about it in the comments.
