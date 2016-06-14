---
layout: post
title:  "Backtracking Parser"
date:   2016-06-14 10:16:36 +0530
categories: compiler
comments: true
page_id: backtrackingparser
---
When LL(K) parsing is not enough, our next step is to extend into Backtracking parser. Backtracking parsers allow arbitrary lookahead letting us parse more complicated languages which look similar from left edge.

Our strategy here is to speculatively attempt to parse alternatives in order until we find the match, although the parsing mechanism used here is similar to LL(K) top down parsing instead of throwing error when we can't match a particular alternative we rewind and try next alternative. This is pretty similar to how one would find a path in a maze using back tracking strategy. When we found a speculative match we can start doing the real matching. 

{% gist 7a2595d4a391aff03c3e0f09a2d13af9 %}

And these speculative predicates can easy be implemented by trying to match related alternative in a try-catch block as in *speculateFuncDefinition*.

