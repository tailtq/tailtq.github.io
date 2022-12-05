---
title: Modify Facebook's Duckling
author: Tai Le
date: 2022-11-04
tags: [Haskell, Back-end]
---

It has been a long time since I wrote my last post, which is about the [Bridge Pattern](posts/how-i-reduced-rasa-testing-time-by-60-80/), I have been quite busy and my mind has been stuffed with many things lately so I hardly produce any posts, so sorry...

The idea of writing this new blog post popped into my mind spontaneously when I completed one task in my company: adding more patterns to the DateTime parser library. I believe it is a good idea because it would help remind my future self, who is having a hard time with himself, that he can accomplish things, even if they are complicated and require many skills.

The Facebook library is [Duckling](https://github.com/facebook/duckling), it is a Haskell library that parses text into structured data. Here is an example showing what the engine can do:

```shell
# From Duckling GitHub repository
"the first Tuesday of October"
=> {"value":"2017-10-03T00:00:00.000-07:00","grain":"day"}

"80F"
=> {"value":80,"type":"value","unit":"fahrenheit"}
```

In general, the library contains a bunch of Regex patterns and rules that help us classify the category of the free-text (datetime, number, phone numbers), then automatically extract the information from that. Back then, my mission was to add some rules to the library because some messages couldn't be understood. Below is my approach to solving it.


## 1. Approach

Because the library is written in Haskell, so I needed to learn the language first. At that time, I had two options:

1. Learn the whole language and know every syntax and how to set up servers
2. Just learn enough to perform the task

From my perspective, I rarely use Haskell after completing the task and I still have other things to learn, so I chose to stick with the second option. Besides, I usually favor the second option for other technologies and it's perfectly reasonable to call me a hasty or practical guy, but I believe it would be a waste of time to learn something in-depth and eventually throw it away. It is fine to learn the basics and dive in depth later if we need to.

After learning Haskell, I would clone the source code, build it, and test several functions. Subsequently, I would read it and understand its structure to add the new code in the ways that I want.


## 2. Implementation

### a. Haskell

I haven't delved into functional programming yet but as far as I know, Haskell is the one; we only need to describe what tasks should be done and leave the responsibility to computers to handle them. Subsequently, computers will choose the most effective way to process tasks. The abstracted layers are often complex and require a great understanding of many aspects of Computer Science to work on.

To grab the syntax of Haskell, I used two resources:
- [Learn X in Y minutes](https://learnxinyminutes.com/docs/haskell/): This one summarizes all syntax with clear explanations.
- [Haskell Tutorial](https://www.youtube.com/watch?v=02_H3LjqMr8&ab_channel=DerekBanas) by [Derek Banas](https://www.youtube.com/c/derekbanas): This one also walks through Haskell's syntax, but it allows me to code along and this is how I learn.

After using them, I learned plenty of data structures, data types, how to define a function, etc. In conclusion, it took me 6-8 hours to learn the language at the basic level and to set up the run-time environment for Haskell locally.


### b. Duckling

Duckling structure is very easy to understand even for a guy having a limited understanding of Haskell like me. That's why I always imagine guys in FANG or other big corporations are so intelligent, they can write clean and organized code for the community. If the source wasn't Duckling, I would have experienced a hard time understanding and modifying the code.

During the journey, I set up the server and called several API requests to ensure it worked fine. Subsequently, in the Extending Duckling part on GitHub, it told me that I only needed to modify 2 files:

1. `Duckling/<Dimension>/<Lang>/Rules.hs`
2. `Duckling/<Dimension>/<Lang>/Corpus.hs`

**Note:**
- Dimension here is the category of text, such as Distance, PhoneNumber, Email,...
- The `Rules.hs` file contains the rules and patterns that parse free texts into structured data.
- The `Corpus.hs` file contains the test cases to verify the rules.

Therefore, I could skip other non-related parts such as building API using Haskell. Based on the task I needed to do, the dimension is Time and the language is EN (English). Therefore, the files that we need to modify would be available in the `Duckling/Time/EN` directory. Note that the country code folders contain the rules respected to each country, such as `Cyber Monday in the US` or `Read Across America Day`.

A rule contains 3 parts:
- `name`: Describe what the rule will catch (an illustration of the pattern).
- `pattern`: Specify the underlying pattern for the rule.
- `prod`: Receive the pattern's output and handle the logic of parsing the text.

Below are the data types that we need to understand before diving deep into any particular rules:

```haskell
-- Production: A function that takes a list of tokens, parses and map them into another format, and return the formatted data (one token).
type Production = [Token] -> Maybe Token
-- Predicate: A function that takes one token and check if its format is valid (check valid day of week, day of month, year, etc...).
type Predicate = Token -> Bool
-- 
data PatternItem = Regex PCRE.Regex | Predicate Predicate

type Pattern = [PatternItem]

data Rule = Rule
  { name :: Text
  , pattern :: Pattern
  , prod :: Production
  }
```

Below is one particular rule, it matches the text having the pattern `this|next <day-of-week>` and returns structured data after parsing:

```haskell
-- Describe the data type of the variable
ruleNextDOW :: Rule
-- Define an object based on data Rule (like an object with respective properties)
ruleNextDOW = Rule
  { name = "this|next <day-of-week>"
  , pattern =
    [ regex "(this|next)"  -- If the text matches this pattern -> pass
    , Predicate isADayOfWeek  -- If the text matches mon, monday, tue, tues, ... -> pass
    ]
  , prod = \case
      (
        Token RegexMatch (GroupMatch (match:_)):
        Token Time dow:
        _) -> do
          td <- case Text.toLower match of
                  "this" -> Just $ predNth 0 True dow
                  "next" -> intersect dow $ cycleNth TG.Week 1
                  _ -> Nothing
          tt td
      _ -> Nothing
  }
```

For example, today is Saturday November 19th; if I input `this sunday`, I will retrieve `November 20th`.

```shell
>> curl -XPOST http://0.0.0.0:8000/parse --data 'locale=en_GB&text=this sunday'      
[{"body":"this sunday","start":0,"value":{"values":[{"value":"2022-11-20T00:00:00.000-08:00","grain":"day","type":"value"}],"value":"2022-11-20T00:00:00.000-08:00","grain":"day","type":"value"},"end":11,"dim":"time","latent":false}]
```

As my little research told me, `\case do` syntax allows matching valid text, executing the code block, and ignoring other invalid cases. Note that `:` is used to separate the variables in `\case do`, `_` represents things we don't use and there must be one after listing all variables. Inside the pattern, I believe that it uses the high-order function concept in functional programming to convert a data type to another:

1. First variable: convert objects having the `Token` data type to `RegexMatch` which can parse into string(s).
2. Second variable: convert objects having the `Token` data type to `Time`, the converted object can use properties of `TimeData` because those two are mapped in `Ducking/Types.hs` file.

For now, our Sunday is converted into structured data containing multiple Sundays. Subsequently, in the code block, it uses `\case do` again for each text. In `this` case, firstly call the function `predNth` with 3 arguments `0`, `True`, `dow` to select the Sunday in this week; then it retrieves the concrete value from `Maybe` (like `Optional` in Python) returned from `predNth`. In `next` case, it simply chooses the intersection between multiple Sundays and the next week's time.

Eventually, it converts the final daytime to a token using `tt` function. That's how a rule works.


### c. Custom rules

And here are the rules that I wrote, I don't know why Duckling doesn't support these cases though. I won't walk you guys through these anymore because they are quite similar to the one above. The key here is to read other rules to understand the `pattern` and `prod`, then we can implement our own rules.

__Rule 1:__ The first one is simply just a day of the week and its date but Ducking assumes the date is the year.

```haskell
-- Monday 26
ruleDOWDOM :: Rule
ruleDOWDOM = Rule
  { name = "<day-of-week> <day-of-month> (ordinal or number)"
  , pattern =
    [ Predicate isADayOfWeek
    , Predicate isDOMValue
    ]
  , prod = \case
      (Token Time dow:dom:_) -> do
        d <- getIntValue dom
        Token Time <$> intersect dow (dayOfMonth d)
      _ -> Nothing
  }
```

Test:

```shell
>> curl -XPOST http://0.0.0.0:8000/parse --data 'locale=en_GB&text=tues 22'  
[{"body":"tues 22","start":0,"value":{"values":[{"value":"2022-11-22T00:00:00.000-08:00","grain":"day","type":"value"},{"value":"2023-08-22T00:00:00.000-07:00","grain":"day","type":"value"}],"value":"2022-11-22T00:00:00.000-08:00","grain":"day","type":"value"},"end":7,"dim":"time","latent":false}]
```

__Rule 2:__ Ducking misrecognizes the number behind `#` as a date in a month, so adding another rule is appropriate.

```haskell
-- #1 Thursday, September 15 at 10:00am
ruleOrderDOWMonthDOMTime :: Rule
ruleOrderDOWMonthDOMTime = Rule
  { name = "#1 <day-of-week>, <named-month> <day-of-month> (ordinal or number) at hh:mm am|pm"
  , pattern =
    [ regex "#?\\d+.?"
    , Predicate isADayOfWeek
    , regex ",| +"
    , Predicate isAMonth
    , Predicate isDOMValue
    , regex "(?:at +)?((?:[01]?\\d)|(?:2[0-3]))[:.]([0-5]\\d) ?(?:([ap])\\.?m?\\.?)?"
    ]
  , prod = \case
      (
        _:
        Token Time dow:
        _:
        Token Time td:
        dom:
        Token RegexMatch (GroupMatch (hh:mm:ap:_)):
        _) -> do
        d <- getIntValue dom
        h <- parseInt hh
        m <- parseInt mm

        hm <- case Text.toLower ap of
          "a" -> Just $ timeOfDayAMPM True (hourMinute True h m)
          "p" -> Just $ timeOfDayAMPM False (hourMinute True h m)
          _ -> Just $ hourMinute False h m
        dowHm <- intersect dow hm
        domDowHm <- intersect (dayOfMonth d) dowHm
        Token Time <$> intersect td domDowHm
      _ -> Nothing
  }
```

Test:

```shell
>> curl -XPOST http://0.0.0.0:8000/parse --data 'locale=en_GB&text=#1. Thur September 15 13:30'  
[{"body":"#1. Thur September 15 13:30","start":0,"value":{"values":[{"value":"2022-09-15T13:30:00.000-07:00","grain":"minute","type":"value"},{"value":"2016-09-15T13:30:00.000-07:00","grain":"minute","type":"value"}],"value":"2022-09-15T13:30:00.000-07:00","grain":"minute","type":"value"},"end":27,"dim":"time","latent":false}]
```

## 3. Conclusion

Recently, I have modified quite a lot libraries to add more features and improve performance. Luckily, the approach I used is time-effective and doesn't require a huge effort. It has some drawbacks which are not understanding the technology deeply and sometimes wearing away our patience. In this case, even though I performed the task pretty well, I don't have the knowledge to write a simple function, let alone building servers, ... Therefore, if you need to use any technology in the long-term, consider delving in-depth.
