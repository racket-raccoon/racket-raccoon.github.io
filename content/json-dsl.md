+++
title = "When DSLs Shine: Processing JSON in Racket and Beyond"
date = 2025-02-01
[taxonomies]
tags = ["json", "dsl", "I/O"]
+++

I wanted to get my Garmin step counts into a spreadsheet, so I wrote a small Racket script.
Then somebody in Janet Carr's chat pointed me at jsonquerylang, which made the same job almost embarrassingly small.

<!-- more -->

Recently, I needed to extract my step data collected by my watch from the Garmin website and put it into a spreadsheet.
I decided this would be a good opportunity to solve the problem with a little Racket script.

Fortunately, the web developer tools in my browser were able to extract the step data in JSON format.

The file format looks something like this:

```json
{
    "values":
    [
        {
            "calendarDate":"2024-12-31",
            "values":{
                "stepGoal":8000,
                "totalSteps":9688,
                "totalDistance":7847
            }
        },
        {
            "calendarDate":"2025-01-01",
            "values":{
                "stepGoal":8000,
                "totalSteps":8150,
                "totalDistance":6914
            }
        }
    ]
}
```

I only needed dates and step counts, and I was specifically interested in dates starting from January 14.
While the entries appeared to be sorted already, I decided to apply sorting just to be safe.
After processing, I wanted to save the steps into a CSV file that could be easily imported by any spreadsheet software.

My first attempt looked something like this:

```Racket
#lang racket

(require json)

(define array
  (hash-ref
   (call-with-input-file "steps.json" read-json)
   'values))

(define date->steps
  (for/list ([val array])
    (cons (hash-ref val 'calendarDate)
          (hash-ref (hash-ref val 'values) 'totalSteps))))

(define data
  (sort
   (filter (λ (rec) (string>=? (car rec) "2025-01-14")) date->steps)
   string<?
   #:key car))

(call-with-output-file "steps.csv"
  #:exists 'replace
  (λ (out)
    (display (string-join (map number->string (map cdr data)) ",") out)))
```

Using `call-with-input-file` and `call-with-output-file` instead of working with file ports directly helps us to close those ports automatically as soon as we're done with our I/O operations.
The `read-json` function is defined in the `json` [library](https://docs.racket-lang.org/json/index.html "docs"), which is part of the `base` package and available out of the box.

The parsed output looks like this:

```Racket
'#hasheq((values
          .
          (#hasheq((calendarDate . "2024-12-31")
                   (values
                    .
                    #hasheq((stepGoal . 8000)
                            (totalDistance . 7847)
                            (totalSteps . 9688))))
           #hasheq((calendarDate . "2025-01-01")
                   (values
                    .
                    #hasheq((stepGoal . 8000)
                            (totalDistance . 6914)
                            (totalSteps . 8150)))))))
```

A JSON object is represented with `hasheq` using symbols as keys, while a JSON array becomes a list.
Fortunately, the dates are formatted as *YYYY-MM-DD*, meaning that regular lexicographical sorting will also maintain chronological order.
Finally, we form a comma-separated string and write it to a file using the `display` function.
The `#:exists` flag specifies how to handle situations when the output file already exists.
By default, an exception would be thrown, so it's easier to just silently override the file each time.

I was working on this small exercise while hanging out on [Janet's stream](https://www.twitch.tv/janetacarr/ "Janet Carr") (amazing Clojure developer) when someone in the chat mentioned [jsonquerylang](https://jsonquerylang.org/ "playground") - a very cute and impressive DSL for JSON processing.
And what's a Racket blog without a DSL discussion?
Just look at how cleanly the exact same idea can be expressed in this language:

```
.values
  | pick(.calendarDate, .values.totalSteps)
  | filter(.calendarDate >= "2025-01-14")
  | sort(.calendarDate)
  | map(.totalSteps)
  | join(",")
```

There's a live playground at [https://jsonquerylang.org/](https://jsonquerylang.org/ "playground") where you can simply paste your JSON into the "Input" text area and edit the query line by line, seeing the query result updated live in the output field.
You can tell how well this small language is designed by how quickly I was able to write this query without any prior experience, guided only by examples and quick reference.

I don't plan to replace every ten-line Racket script with a DSL, but this one fit the problem unusually well.
Its operations matched the questions I was already asking of the JSON, and I managed to write the query from the examples and quick reference alone.
