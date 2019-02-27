---
layout: post
title: Calculating Frequency Distribution in PostgreSQL
category: tutorials
tags:
- PostgreSQL
- Statistics
---

I've always loved statistics. Being a software engineer has allowed me a lot of opportunities
to solve complex statistical problems in SQL. There are lots of tools optimized for running statistical analysis such as [SPSS](https://www.ibm.com/analytics/spss-statistics-software) or [R](https://www.r-project.org/about.html), but these are generally used as retrospective tools to look back at a 
dataset. With programming, we want to consider the current state of our data which
means running calculations on demand. Recently, I had the opportunity to calculate
frequency distribution in SQL.
<!--excerpt-->

## Basic Frequency Distribution

<blockquote class="Info Info-right"><strong>Frequency Distribution</strong><br />
  "A frequency (distribution) table shows the different measurement categories and the number of observations in each category."
<cite><a href="https://www.ncbi.nlm.nih.gov/pmc/articles/PMC3117575/">- PubMed Central</a></cite>
</blockquote>

Think about a professor giving a multiple choice exam. There are a set number of options: a, b, c, and d
listed for each question and a set number of students taking the exam. **Frequency
distribution is calculating how many students selected a, b, c, or d as their answer.**

You can think of each option (a, b, c, d) as a different category we'd like
to group students into based on their answer. For the following example, we'll
assume that we're talking about a single exam:

{% highlight sql %}
SELECT 
  student_answers.choice, 
  COUNT(student_answers.choice) as total_answers
FROM student_answers
GROUP BY student_answers.choice
ORDER BY student_answers.choice -- To ensure the order is a, b, c, d
{% endhighlight %}

The exam might have a distribution that looks like:

| choice        | total_answers    |
|---------------|------------------|
| a             |   25            |
| b             |   50            |
| c             |   30            |
| d             |   70            |

This is the most basic example of a frequency distribution table. Pretty straight forward right? However, the above table only illustrates what the
selected answers were for **all questions** on the exam. Ready for your next
requirement?

## Frequency Distribution for two categories

Now the professor would like to know frequency distribution on a more
granular level. Specifically, given an ExamQuestion how many students answered
a, b, c, or d? We now have two categories (or groups): **selected answer** and **exam question**.

{% highlight sql %}
SELECT 
  exam_questions.question_text,
  student_answers.choice, 
  COUNT(student_answers.choice) as total_answers
FROM student_answers
INNER JOIN exam_questions ON exam_questions.id = student_answers.exam_question_id
GROUP BY 
  student_answers.choice, 
  exam_questions.question_text
ORDER BY student_answers.choice
{% endhighlight %}

For example purposes, let's assume that the Exam only has three questions: 
1. *What is your name?*
2. *What is your quest?*
3. *What is the airspeed velocity of an unladen swallow?*

Since we're grouping the results by `student_answers.choice` and `exam_questions.question_text`, we end
up with multiple results per question text (1 per choice / question pair)

| Question text | choice        | total_answers    |
|--------------------------------|---------------|------------------|
| "What is your name?" | a             |   12            |
| "What is your name?" | b             |   20            |
| "What is your name?" | c             |   10            |
| "What is your name?" | d             |   5            |
| "What is your quest?" | a             |   12            |
| "What is your quest?" | b             |   5            |
| "What is your quest?" | c             |   15            |
| "What is your quest?" | d             |   40            |
| "What is the airspeed velocity of an unladen swallow?" | a             |   1            |
| "What is the airspeed velocity of an unladen swallow?" | b             |   25            |
| "What is the airspeed velocity of an unladen swallow?" | c             |   5            |
| "What is the airspeed velocity of an unladen swallow?" | d             |   25            |

This is starting to look like useful data.

## Frequency Distribution with Descriptive Statistics

<blockquote class="Info Info-right clearfix"><strong>What is Descriptive Statistics</strong><br />
  "Descriptive statistics allow you to characterize your data based on its properties. 
  There are four major types of descriptive statistics: Measures of Frequency, 
  Measures of Central Tendency, Measures of Dispersion or Variation, Measure of Position"
<cite><a href="https://baselinesupport.campuslabs.com/hc/en-us/articles/204305665-Types-of-Descriptive-Statistics">- Campuslabs.com</a></cite>
</blockquote>

The professor says this looks great but she'd love to know more about
the data involving whether or not the student selected the **correct answer**. Here's the
new requirements:

1. What is the total number of correct answers per question?
2. What is the average number of correct answers per question?

Questions like that are what is referred to as **Descriptive Statistics**. Gathering 
a count of the total number of correct answers is a *Measure or Frequency* while 
average (also known as the mean) number of correct answers is a *Measure of 
Central Tendency*.

Now for a curve ball, she'd like to have only 1 row per question text so that 
the data is easier to read. 

{% highlight sql %}
SELECT 
  inner_query.question_text, 
  SUM(inner_query.total_correct_answers) as total_correct_answers, 
  (SUM(inner_query.total_correct_answers) / SUM(inner_query.total_answers)) * 100 as average_correct_answer
FROM (
  SELECT 
    exam_questions.question_text,
    student_answers.choice, 
    COUNT(student_answers.choice) as total_answers
    COUNT(student_answers.choice) FILTER (WHERE exam_questions.correct_choice = student_answers.choice) as total_correct_answers
  FROM student_answers
  INNER JOIN exam_questions ON exam_questions.id = student_answers.exam_question_id
  GROUP BY 
    student_answers.choice, 
    exam_questions.question_text
  ORDER BY student_answers.choice
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
to calculate an aggregate based on a group of multiple aggregates. 

More specifically, calculations are accomplished by first counting the results within 
the inner query giving us the total answers for each category (as seen in the previous table).
Then `SUM ()` is used to calculate the running total for all categories. If we were
to calculate this manually for the question, "What is your name?" it would look like this 
for the inner query:

| Question text | choice        | total_answers    |
|--------------------------------|---------------|------------------|
| "What is your name?" | a             |   12            |
| "What is your name?" | b             |   20            |
| "What is your name?" | c             |   10            |
| "What is your name?" | d             |   5            |

Then the outer query aggregates each column into a SUM of all the existing
aggregates given a single question.

`12 + 20 + 10 + 5 = 20`

| Question text | total_correct_answers   |
|---------------|-------------------------|
| "What is your name?" | 20             |

For calculating average, the SUM of the total correct answers is divided by the SUM of the total answers. 
We multiply by 100 to convert it to a percentage.

Below is the end result given the newest SQL changes above.

| Question text | total_correct_answers   | average_correct_answer    |
|---------------|-------------------------|---------------------------|
| "What is your name?" | 20             |   42.5532            |
| "What is your quest?" | 40             |   55.5555            |
| "What is the airspeed velocity of an unladen swallow?" | 1             |   1.78571            |

This is looking really good. Unfortunately, by running the aggregate for
total correct and average correct, we lost the frequency distribution for 
each question. It's a bit tricky to get this back but possible.

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

Just like we did above while calculating **total_correct_answers** and **average_correct_answer**, 
the aggregate of the inner query is utilized as an aggregate of the outer query.

{% highlight sql %}
SELECT 
  inner_query.question_text, 
  SUM(inner_query.total_correct_answers) as total_correct_answers, 
  (SUM(inner_query.total_correct_answers) / SUM(inner_query.total_answers)) * 100 as average_correct_answer,
  jsonb_agg(inner_query.frequency_distribution) as frequency_distribution_json
FROM (
  SELECT 
    exam_questions.question_text,
    student_answers.choice, 
    COUNT(student_answers.choice) as total_answers
    COUNT(student_answers.choice) FILTER (WHERE exam_questions.correct_choice = student_answers.choice) as total_correct_answers,
    jsonb_build_object('choice', student_answers.choice, 'frequency', COUNT(student_answers.choice)) as frequency_distribution
  FROM student_answers
  INNER JOIN exam_questions ON exam_questions.id = student_answers.exam_question_id
  GROUP BY 
    student_answers.choice, 
    exam_questions.question_text
  ORDER BY student_answers.choice
) inner_query
GROUP BY inner_query.question_text
{% endhighlight %}

| Question text | total_correct_answers   | average_correct_answer    | frequency_distribution_json |
|---------------|-------------------------|---------------------------|-----------------------------|
| "What is your name?" | 20             |   42.5532            | `[{ "choice": "a", "frequency": 12 }, { "choice": "b", "frequency": 20 }, { "choice": "c", "frequency": 10 }, { "choice": "d", "frequency": 5 }]` |
| "What is your quest?" | 40             |   55.5555            | `[{ "choice": "a", "frequency": 12 }, { "choice": "b", "frequency": 5 }, { "choice": "c", "frequency": 15 }, { "choice": "d", "frequency": 40 }]` |
| "What is the airspeed velocity of an unladen swallow?" | 1             |   1.78571            | `[{ "choice": "a", "frequency": 1 }, { "choice": "b", "frequency": 25 }, { "choice": "c", "frequency": 5 }, { "choice": "d", "frequency": 25 }]` |

Success! With that being our final modification to the query, we now have a table that contains:
1. The total correct answers per question
2. The average correct answer per question
3. A frequency distribution per question

So what does the inner query look like with the new `jsonb_build_object()` method?

If we were to back track to the table that contained a result per question per 
choice and look at what the following SQL returns it would look like this:

{% highlight sql %}
  jsonb_build_object('choice', student_answers.choice, 'frequency', COUNT(student_answers.choice)) as frequency_distribution
{% endhighlight %}

| Question text | choice        | total_answers    | frequency_distribution |
|--------------------------------|---------------|------------------|------------------|
| "What is your name?" | a             |   12            | `{ "choice": "a", "frequency": 12 }` |
| "What is your name?" | b             |   20            | `{ "choice": "b", "frequency": 20 }` |
| "What is your name?" | c             |   10            | `{ "choice": "c", "frequency": 10 }` |
| "What is your name?" | d             |   5            | `{ "choice": "d", "frequency": 5 }` |
| "What is your quest?" | a             |   12            | `{ "choice": "a", "frequency": 12 }` |
| "What is your quest?" | b             |   5            | `{ "choice": "b", "frequency": 5 }` |
| "What is your quest?" | c             |   15            | `{ "choice": "c", "frequency": 15 }` |
| "What is your quest?" | d             |   40            | `{ "choice": "d", "frequency": 40 }` |
| "What is the airspeed velocity of an unladen swallow?" | a             |   1            | `{ "choice": "a", "frequency": 1 }` |
| "What is the airspeed velocity of an unladen swallow?" | b             |   25            | `{ "choice": "b", "frequency": 25 }` |
| "What is the airspeed velocity of an unladen swallow?" | c             |   5            | `{ "choice": "c", "frequency": 5 }` |
| "What is the airspeed velocity of an unladen swallow?" | d             |   25            | `{ "choice": "d", "frequency": 25 }` |

The magic of this happens in the outer query with `jsonb_agg()` which
aggregates all the inner frequency distribution columns into an array
of jsonb key-value pairs like we see in the above final query. This allows
for both Descriptive Statistics on the entire dataset while providing Frequency Distribution
for a single question.

## Potential alternative

An alternative that I didn't dig into is called `width_bucket()`. These allow you
to specify a column that has a minimum and maximum range and then split it into
buckets (again this is the same as groups or categories). For more information on
`width_bucket()` [check out the following documentation](https://www.postgresql.org/docs/9.1/functions-math.html).

Got a different SQL statistics trick? How about an improved approach to the above? I'd love to hear about it in the comments.
