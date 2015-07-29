---
layout: post
title:  "Speeding up edX Unit Tests"
date:   2015-07-26 11:16:05
categories: tests, performance
---

Slow tests have consistently been cited as one of the biggest pain points for
Open edX developers. The long straw at the moment are unit tests. Running the
full set takes about an hour and fifteen minutes of total time on Jenkins
(though we shard it out, so the actual turnaround time is in the 45-50 minute
range). For our most recent hackathon, I wanted to see what I could do to get
those numbers down.


### Starting Out

Performance optimization is a delightful and deep subject, but the basic version
always looks the same:

1. Find the slow thing.
2. Make it faster [and/or...]
3. Do it less often.
4. Repeat.

The fact that we're talking about test code introduces interesting constraints:

* Results must be consistent regardless of which tests are run or their ordering.
* This means side-effects (e.g. database operations) should be cleaned up
  between tests.
* Query counting tests must remain predictable.
* We're ideally looking for something that will have a substantial effect across
  a very wide area of code as opposed to narrowly optimizing a vertical slice.
* The test infrastructure should be modified to encourage faster tests by
  default.

### Finding the Problem

As always, getting measurements is the first priority. Our intuition is almost
always wrong. Running profiling on the entire set of unit tests didn't work out
very well for me. It took 
