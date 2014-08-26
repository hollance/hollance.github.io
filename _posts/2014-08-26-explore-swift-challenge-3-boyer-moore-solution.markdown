---
layout: post
title: "Explore Swift Challenge #3: Boyer-Moore Solution"
date: 2014-08-26 12:23:00
tags: [swift]
---
@Eiam8821 has been running the [Explore Swift Challenge](https://github.com/Eiam8821/Explore-Swift-Challenge) as a way to explore what it means to write Swift code and how we can solve actual programming problems in a true Swift-like manner. (See also [Reddit](http://www.reddit.com/r/swift/comments/2c67rd/swift_challenge_explore_what_it_means_to_write/).)

Challenge #3 goes as follows:

> Implement a `findIndexOf()` extension on `String`.
> `findIndexOf()` returns the `String.Index?` of the starting character in the supplied `String` 
  
For example:

{% highlight swift %}
// Input: 
var boring = "Hello, World"
boring.findIndexOf("World")

//Output:
<String.Index?> 7

//Input:
let clarus = "🐶🐮"
clarus.findIndexOf("🐮")
	   
//Output:
<String.Index?> 2
{% endhighlight %}

As with the other challenges, the idea is to write the solution in pure Swift. You can't import Cocoa, Foundation, AppKit, or UIKit.

First, a brute-force solution:

{% highlight swift %}
extension String {
  func findIndexOf(pattern: String) -> String.Index? {
    for i in self.startIndex ..< self.endIndex {
      var j = i
      var found = true
      for p in pattern.startIndex ..< pattern.endIndex {
        if j == self.endIndex || self[j] != pattern[p] {
          found = false
          break
        } else {
          j = j.successor()
        }
      }
      if found {
        return i
      }
    }
    return nil
  }
}
{% endhighlight %}

This looks at each character in the source string in turn. If the character equals the first character of the search pattern, then it enters the inner loop that checks whether the rest of the pattern matches. If no match is found, the outer loop continues where it left off. This repeats until a complete match is found or the end of the source string is reached.

The brute-force approach works OK, but it's not very efficient (or pretty). As it turns out, you don't have to look at each character from the source string -- you can often skip ahead multiple characters.

That skip-ahead algorithm is called [Boyer-Moore](https://en.wikipedia.org/wiki/Boyer–Moore_string_search_algorithm) and it has been around for a long time. There are probably faster string search algorithms out there, but Boyer-Moore is no slouch either.

Here's how you'd write it in Swift:

{% highlight swift %}
extension String {
  func findIndexOf(pattern: String) -> String.Index? {
    // Cache the length of the search pattern because we're going to 
    // use it a few times and it's expensive to calculate.
    let patternLength = countElements(pattern)
    if patternLength == 0 {
      return nil
    }

    // Make the skip table. This table determines how many times successor() 
    // needs to be called when a character from the pattern is found.
    var skipTable = [Character: Int]()
    for (i, c) in enumerate(pattern) {
      skipTable[c] = patternLength - i - 1
    }

    // This points at the last character in the pattern.
    let p = pattern.endIndex.predecessor()

    // The pattern is scanned right-to-left, so skip ahead in the string by
    // the length of the pattern. (Minus 1 because startIndex already points
    // at the first character in the source string.)
    var i = advance(self.startIndex, patternLength - 1)

    // Keep going until the end of the string is reached.
    while i < self.endIndex {

      // Does the current character match the last character from the pattern?
      if self[i] == pattern[p] {

        // There is a possible match. Do a brute-force search backwards.
        var j = i
        var q = p
        var found = true
        while q != pattern.startIndex {
          j = j.predecessor()
          q = q.predecessor()
          if self[j] != pattern[q] {
            found = false
            break
          }
        }

        // If the pattern matches, we're done. If no match, then we can only
        // safely skip one character ahead.
        if found {
          return j
        } else {
          i = i.successor()
        }
      } else {
        // The characters are not equal, so skip ahead. The amount to skip is 
        // determined by the skip table. If the character is not present in the
        // pattern, we can skip ahead by the full pattern length. But if the 
        // character *is* present in the pattern, there may be a match up ahead 
        // and we can't skip as far.
        i = advance(i, skipTable[self[i]] ?? patternLength)
      }
    }
    return nil
  }
}
{% endhighlight %}

This code is based on the article ["Faster String Searches" by Costas Menico](http://www.drdobbs.com/database/faster-string-searches/184408171)
from Dr Dobb's magazine, July 1989. (Yes, 1989! Sometimes it's useful to keep those old magazines around.)

It works as follows. You line up the search pattern with the source string and see what character matches the *last* character of the search pattern:

	source string:  Hello, World
	search pattern: World
	                    ^

There are three possibilities:

1. The two characters are equal. You've found a possible match.

2. The characters are not equal, but the source character does appear in the search pattern elsewhere.

3. The source character does not appear in the search pattern at all.

In the example, the characters `d` and `o` do not match, but `o` does appear in the search pattern. That means we can skip ahead several positions:

	source string:  Hello, World
	search pattern:    World
	                       ^

Note how the two `o` characters line up now. Again you compare the last character of the search pattern with the search text: `d` vs `W`. These are not equal but the `W` does appear in the pattern. So we skip ahead again to line up those two `W` characters:

	source string:  Hello, World
	search pattern:        World
	                           ^

This time the two characters are equal and there is a possible match. To verify the match you do a brute-force search, but backwards, from the end of the search pattern to the beginning. And that's all there is to it.

The amount to skip ahead at any given time is determined by the "skip table", which is a dictionary of all the characters in the search pattern and the amount to skip by. The skip table in the example looks like:

	W: 4
	o: 3
	r: 2
	l: 1
	d: 0

The closer a character is to the end of the pattern, the smaller the skip amount. If a character appears more than once in the pattern, the one nearest to the end of the pattern determines the skip value for that character.

Note that if the pattern is just one character, it's faster to do a brute-force search. There's probably a trade-off between the time it takes to build the skip table and doing brute-force for short patterns.

The code appears to work well, but I don't really like the following bit:

{% highlight swift %}
var j = i
var q = p
var found = true
while q != pattern.startIndex {
  j = j.predecessor()
  q = q.predecessor()
  if self[j] != pattern[q] {
    found = false
    break
  }
}

if found {
  . . .
{% endhighlight %}

This performs the backwards search that happens when the last character from the search pattern matches the character from the source string. We need to step backwards through both strings until we find a character that doesn't match, or until we've reached the beginning of the search pattern. 

I wonder if there is a more Swift-ish way to write this...

The following would work:

{% highlight swift %}
let j = advance(i, -patternLength + 1)
if self[j...i] == pattern {
  return j
} else {
  i = i.successor()
}
{% endhighlight %}

This steps back through the source string until where the pattern starts, and then compares the substring to the search pattern. The code is a lot shorter but it may be slower because it potentially does more work.
