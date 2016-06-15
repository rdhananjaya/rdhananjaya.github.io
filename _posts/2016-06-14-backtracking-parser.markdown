---
layout: post
title:  "Backtracking Parser"
summary: "Backtracking parsers allow arbitrary lookahead letting us parse more complicated languages which look similar from left edge."
date:   2016-06-14 10:16:36 +0530
categories: compiler
comments: true
page_id: backtrackingparser
---
When LL(K) parsing is not enough, our next step is to extend into Backtracking parser. Backtracking parsers allow arbitrary lookahead letting us parse more complicated languages which look similar from left edge.

Our strategy here is to speculatively attempt to parse alternatives in order until we find the match, although the parsing mechanism used here is similar to LL(K) top down parsing instead of throwing error when we can't match a particular alternative we rewind and try next alternative. This is pretty similar to how one would find a path in a maze using back tracking strategy. When we found a speculative match we can start doing the real matching. 

{% gist 7a2595d4a391aff03c3e0f09a2d13af9 %}

And these speculative predicates can easy be implemented by trying to match related alternative in a try-catch block as in *speculateFuncDefinition*. By using try-catch error handling mechanism to detect speculation miss-match we can reuse our usual LL(K) parser code after a little modification.

*mark()* and *release()* methods pushes current token index into a stack, and pops out and rewind the token stream, respectively. This works nicely until we have some side effecting code inside match functions. To guard against such dual invocation of side effects, when speculating and when matching, we can use a flag to indicate whether this is a speculation or actual matching and only perform the side effecting action when it's appropriate to do so.

Although we can parse complex languages using backtracking parsers, they are inherently inefficient because we have duplicate work, we can use memoizing techniques to avoid this duplicate work by storing information when matching in the speculative phase.
