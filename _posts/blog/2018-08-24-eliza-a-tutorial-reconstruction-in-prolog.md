---
layout: post
title: ELIZA, a tutorial reconstruction in prolog
permalink: /2018/08/24/eliza-a-tutorial-reconstruction-in-prolog.html
categories:
- blog
---

It has been a while since I wrote a blog post.

I’ve been going through the Natural Language Processing for Prolog Programmers
book and recreating [ELIZA](https://en.wikipedia.org/wiki/ELIZA) was one of the 
exercises. I thought that rewriting it might be fun and a perfect opportunity to
write an article for my blog.

This article will assume a workable knowledge of prolog.

ELIZA?
======================

ELIZA was one of the first, and perhaps the most famous, chatter bot.

It worked by following simple rules - first finding keywords in the user input
and then matching patterns related to that keyword (scenario). The choice of the
scenarios and responses gave an illusion of understanding.

A more sophisticated user can make it break quite easily but at the time it
seemed to pass the Turing test.

I based my implementation directly on the original
[aricle](http://web.stanford.edu/class/cs124/p36-weizenabaum.pdf).

The scenarios described in the article seemed not to be easily copy-passable due to
OCR errors and duplicates so I’ve [cleaned it up manually](https://github.com/bartosz-witkowski/books/blob/master/natural-language-processsing-for-prolog-programmers/chapter-02/eliza-script.txt).

Originally, ELIZA was written in a variant of LISP, and it shows - there’s a of
implicit knowledge on how to interpret the data types in the scenarios but
fortunately the article explains that fully.

The full source code for eliza is available 
[here](https://github.com/bartosz-witkowski/books/blob/master/natural-language-processsing-for-prolog-programmers/chapter-02/eliza.pl). 
While I like prolog, I’ve never used it in anger - I missed types a lot hence
runtime tag checking strewn everywhere. At times it was difficult to understand
what the script was doing so I introduced permanent (toggable) debugging in the
form of a `dwrite/2` predicate ([see](https://github.com/bartosz-witkowski/books/blob/master/natural-language-processsing-for-prolog-programmers/chapter-02/eliza.pl#L23-L43)).

I won’t present the entire source code for each step of the article but I’ll
link to the relevant parts where needed.

Preliminaries out of the way, let’s start with the basics.

First steps
===========

The logic of the program is actually pretty simple.

Eliza prints a greeting and runs in an loop. Each time input is entered, the
program splits that into words. Then the words are matched against the keywords
(by a simple membership test) the scripts are sorted by keyword priority and
then their patterns are matched.

Next going by highest priority script first, the input is matched against
patterns in the script and an action that matches the pattern is chosen.

As a start, we will create a program that introduces itself with the special [start script](https://github.com/bartosz-witkowski/books/blob/master/natural-language-processsing-for-prolog-programmers/chapter-02/eliza.pl#L1426), 
prompts for user input, reads it, and does absolutely nothing else.

To handle user input I used the [readatom](https://github.com/bartosz-witkowski/books/blob/master/natural-language-processsing-for-prolog-programmers/chapter-02/readatom.pl) package provided by the author of NLP for Prolog Programmers.

We will use lists of atoms as both input and output. I decided to go with the
original look and feel of ELIZA and capitalize output, this allows me to input
sentences as `[simple, lists, of, atoms]` without having to bother with quoting
anything except punctuation. To show sentences properly we need to capitalize
the atoms, treat punctuation specially as not to write spaces after it.

At this point we defined:
  * the main loop (ignore everything after `read_atomics`)
  * showing the output see [here](https://github.com/bartosz-witkowski/books/blob/master/natural-language-processsing-for-prolog-programmers/chapter-02/eliza.pl#L238-L249), and [here](https://github.com/bartosz-witkowski/books/blob/master/natural-language-processsing-for-prolog-programmers/chapter-02/eliza.pl#L98-L102)

Script handling
===============

Let’s start with the **sorry** script and the catch-all **none** script.

I decided on the following data structure for the scripts:

{% highlight haskell %}
data Script = Script { keyword :: Keyword,
                     , patterns :: List Pattern }
data Keyword = Keyword { word :: String
                       , priority :: Int }
data Pattern = Pattern { matched :: Matched
                       , actions :: List Action }
data Action = Response List String
{% endhighlight %}

We’ll need more `Action` types than just `Response` in the future, so that’s why
it looks that way right now.

All scripts go into the `scripts/1` predicate/table, but I decided against
storing the “special” predicates like `none` there - they’re stored in their own
custom predicates so that their special keywords aren’t matched in the matching
step.

The **sorry** script looks like this:

{% highlight prolog %}
scripts(
  script(
    keyword(sorry, 0),
    [
      pattern(
	matched([_]),
	actions([
	  response([please, do, not, apologize]),
	  response([apologies, are, not, necessary]),
	  response([what, feelings, do, you, have, when, you, apologize, ?]),
	  response([i, have, told, you, that, apologies, are, not, required])
	])
      )
    ]
  )
).
{% endhighlight %}

And the **none** script like this:

{% highlight prolog %}
none_script(
  script(
    keyword(none, 0),
    [
      pattern(
        matched([_]),
        actions([
          response([i, am, not, sure, i, understand, you, fully]),
          response([please, go, on]),
          response([what, does, that, suggest, to, you]),
          response([do, you, feel, strongly, about, discussing, such, things])
        ])
      )
    ]
  )
).
{% endhighlight %}

Of course prolog is
[unityped](https://existentialtype.wordpress.com/2011/03/19/dynamic-languages-are-static-languages/)
and this data structure is just wishful thinking on my part (and many a bug were
created during implementation precisely because the data types inside script
didn’t follow this pattern).

After getting the user input we
[scan](https://github.com/bartosz-witkowski/books/blob/master/natural-language-processsing-for-prolog-programmers/chapter-02/eliza.pl#L528-L551)
the input list for keywords, and sort them by priority.

Then we go by highest priority script first and try to find a match on the
pattern. For the time being matching is just prototyped to match EVERYTHING. So
right now, we can only differentiate between the presence of the **sorry**
keyword and its absence.

We’ll also need a mechanism for storing the last action in each pattern. For
simplicity I decided to use (**script-name**, **pattern-index**,
**action-index**) tuple and assert and retract those as needed in a
`current_action_aux/3` predicate. [Relevant code](https://github.com/bartosz-witkowski/books/blob/master/natural-language-processsing-for-prolog-programmers/chapter-02/eliza.pl#L458-L493)

For ease of use I added a special handling of `quit` during input phase.

ELIZA in action looks like this right now:

```
HOW DO YOU DO. PLEASE TELL ME YOUR PROBLEM
> Sorry for being late.
PLEASE DO NOT APOLOGIZE
> Well, sorry all the same.
APOLOGIES ARE NOT NECESSARY
> Derp.
I AM NOT SURE I UNDERSTAND YOU FULLY
> Herp.
PLEASE GO ON
> quit
true.
```

Matching
============

Next we need need to be able to match more complex patterns than just matching
everything.

Let’s look at the **if** script

```
(IF 3 (
  (0 IF 0) 
  (DO YOU THINK ITS LIKELY THAT 3) 
  (DO YOU WISH THAT 3) 
  (WHAT DO YOU THINK ABOUT 3) 
  (REALLY, 2 3)))
```

Instead of using numbers like in the paper, we’ll use prolog variables. Our translated script looks like:
{% highlight prolog %}
scripts(
  script(
    keyword(if, 3),
    [
      pattern(
        matched([_, if, Y]),
        actions([
          response([do, you, think, it, is, likely, that, Y, ?]) ,
          response([do, you, wish, that, Y, ?]),
          response([what, do, you, think, about, Y, ?]),
          response([really, ',', if, Y, ?])
        ])
      )
    ]
  )
)
{% endhighlight %}

This means that we can’t refer to the “constant” terms (e.g `(Really, 2 3)` must
be translated to `[really, ',', if, Y, ?]`.

To handle all the patterns in the paper we need to match the patterns in the
form of:
```
[Some_Variable]  or
[Some_Variable, atom-or-class-1, atom-or-class-2, ..., atom-or-class-N, OtherVariable] 
```

I’ve made my matching more general so that any combination of atoms and
variables work as long as two variables aren’t next to each other. The
[code](https://github.com/bartosz-witkowski/books/blob/master/natural-language-processsing-for-prolog-programmers/chapter-02/eliza.pl#L365-L435)
is a bit more involved. The purpose of `match_not_var/2` is to handle word
classes and will be explained further along. For now, this can be replaced by
simple unification.

```
HOW DO YOU DO. PLEASE TELL ME YOUR PROBLEM
> If you were an animal, what animal would you be?
DO YOU THINK IT IS LIKELY THAT YOU WERE AN ANIMAL, WHAT ANIMAL WOULD YOU BE ?
> If I were a rich man.
DO YOU WISH THAT I WERE A RICH MAN.
> If I were a wealthy man.
WHAT DO YOU THINK ABOUT I WERE A WEALTHY MAN.
```

Splitting sentences
===================

The above test shows two missing feature of our program - the lack of
translation rules and complex sentences.

The paper states that it scans the input for a comma or a period and if a
keyword is recognized drops the rest:

```
Also, the procedure recognizes a comma or a period as a delimiter. Whenever
either one is encounter and a keyword has already been found, all subsequent
text is deleted from the input message.
```

I’ve replicated this behavior while also taking care of question marks and
semicolons.

Test:
```
HOW DO YOU DO. PLEASE TELL ME YOUR PROBLEM
> If you were an animal, what animal would you be?
DO YOU THINK IT IS LIKELY THAT I WERE AN ANIMAL
> If I were a rich man
DO YOU WISH THAT YOU WERE A RICH MAN
> If I were a wealthy man
WHAT DO YOU THINK ABOUT YOU WERE A WEALTHY MAN
```

Transformations and word classes
================================

Transformations are simple translations of the input text e.g first person
pronouns are exchanged into second person pronouns.

The original paper had a lot of facilities for working with equivalence, classes
and some assorted magic, e.g

```
(you're = I'm ((0 I'm 0) (PRE (I AM 3) = (YOU))))
```

which would replace **you’re** with **i’m** and would basically treat **i am**
as equivalent to **you**.

```
(3) The reassembly rule beginning with the eode "PRE" is encountered
    and the decomposed text reassembled such that the words "I  AM" appear in front
    of the third constituent, determined by the earlier decomposition
```

some other special matching and reassembly rules are:
  * alternatives `(* WANT NEED)`
  * and word classes `(O YOU (/BELIEF) YOU 0)`

Alternatives and word classes are really the same (one requires an explicit
declaration of the words in a class, and the other does not) so I’ll cover them
uniformly with just word classes. For my ELIZA implementation I also won’t
bother with the more complex reassembly rules. This has the unfortunate effect
that we need to be smarter with matching and a few more translation rules than
the original paper. We do need a lot less machinery to get working though.

Translation can be seen [here](https://github.com/bartosz-witkowski/books/blob/master/natural-language-processsing-for-prolog-programmers/chapter-02/eliza.pl#L125-L175),
we capture lists of atoms and translate them to a single atom and a reminder to
be translated. So when matching `you're` or you are we always deal with a
singular atom `'i\'m'`.

Let’s handle word classes. I’ve decided to abuse the dynamic nature of prolog
and introduce a predicate `class/2` into the scripts like so:
{% highlight prolog %}
scripts(
  script(
    keyword(your, 2),
    [
      pattern(
        matched([_, your, _, class(family, F), X]),
        actions([
          response([tell, me, more,about, your, family]),
          response([who, else, in, your, family, X, ?]),
          response([your, F, ?]),
          response([what, else, comes, to, mind, when, you, think, of, your, F, ?])
        ])
      ),
      pattern(
        matched([_, your, X]),
        actions([
          response([your, X, ?]),
          response([why, do, you, say, your, X, ?]),
          response([does, that, suggest, anything, else, which, belongs, to, you, ?]),
          response([is, it, important, to, you, that, your, X, ?])
        ])
      )
    ]
  )
).
{% endhighlight %}

Where family is defined as:

{% highlight prolog %}
family(mother).
family(mom).
family(dad).
family(father).
family(sister).
family(sisters).
family(brother).
family(brothers).
family(wife).
family(husband).
{% endhighlight %}

The matching code needed only a slight modification - instead of unifying the
input with the pattern I needed to introduce the before mentioned
`math_not_var/2` predicate:

{% highlight prolog %}
match_not_var(Input, Pattern) :-
  (  Pattern = class(Predicate, Input)
  -> call(Predicate, Input)
  ;  Input = Pattern
  ).
{% endhighlight %}

The only non-obvious stutter with this minimalistic strategy is the **EVERYONE**
script:

```
(EVERYONE 2 (
    (0 (* EVERYONE EVERYBODY NOBODY NOONE) 0) 
    (REALLY, 2) 
    (SURELY NOT 2) 
    (CAN YOU THINK OF ANYONE IN PARTICULAR) 
    (WHO, FOR EXAMPLE) 
    (YOU ARE THINKING OF A VERY SPECIAL PERSON) 
    (WHO, MAY I  ASK) 
    (SOMEONE SPECIAL PERHAPS) 
    (YOU HAVE A PARTICULAR PERSON IN MIND DON'T YOU) 
    (WHO DO YOU THINK  YOU'RE TALKING ABOUT))) 

(EVERYBODY 2  (= EVERYONE)) 
(NOBODY 2  (=EVERYONE)) 
(NOONE 2  (=EVERYONE)) 
```

which we can fortunately code using normal prolog facilities:
{% highlight prolog %}
everyone(everyone).
everyone(everybody).
everyone(nobody).
everyone('no one').

% ...

scripts(Script) :-
  everyone(Everyone),
  Script = script(
    keyword(Everyone, 2),
    [
      pattern(
        matched([_, Everyone, _]),
        actions([
          response([really, Everyone, ?]), 
          response([surely, not, Everyone, ?]), 
          response([can, you, think, of, anyone, in, particular, ?]),
          response([who, ',', for, example, ?]),
          response([you, are, thinking, of, a, very, special, person]),
          response([who, ',', may, i, ask, ?]),
          response([someone, special, perhaps, ?]),
          response([you, have, a, particular, person, in, mind, 'don\'t', you, ?]),
          response([who, do, you, think, 'you\'re', talking, about, ?])
        ])
      )
    ]
  ).
{% endhighlight %}

Memory
======
Having done that we can proceed to implement eliza’s memory facilities. From the
article:

```
There is, however, another mechanism which causes the system to respond  more
spectacularly in the absence of a key.  The word "MEMORY" is another reserved
pseudo- keyword.  The key list structure associated with it differs  from 
the ordinary one in some  respects.  An example illuminates this point. 

    Consider the following structure:
       (MEMORY MY
          (0 YOUR 0  = LETS DISCUSS FURTHER WHY YOUR 3) 
          (0 YOUR 0  - EARLIER YOU SAID YOUR 3) 

The word "MY" (which must be an ordinary keyword as well) has been
selected to serve a special function.  Whenever it is the highest ranking
keyword of a  text one of the transformations on the MEMORY list is randomly
selected, and a  copy of the text is transformed accordingly.  This
transformation is stored on a first-in-first-out stack for later use.  The
ordinary processes already described are then carried out.  When a text without
keywords is encountered later and a  certain counting mechanism is  in a
particular state and the stack in question is not empty, then the transformed
text is printed out as the reply. It is, of course, also deleted from the stack
of such trans- formation
```

Conceptually this is very simple, the **my** keyword is blessed and each time it
it has the highest priority a random transformation (originally stored under the
blessed **memory** script but stored as a separate `memory_patterns` predicate
in my implementation) is applied to the user input and the result of that
transformation is stored in a FIFO queue.

The original paper didn’t go into much depth WHEN the memory store was accessed:
so I decided to go the easy way and skip the “certain counting mechanism” and
just use the memory whenever it is available.

The [implementation](https://github.com/bartosz-witkowski/books/blob/master/natural-language-processsing-for-prolog-programmers/chapter-02/eliza.pl#L203-L232)
is quite straightforward, I just `assert`/`retract` a `memory/1` predicate on
executing the **my** keyword and similarly `assert`/`retract` a shortened list
after executing the `memory` keyword.

After implementing the memory facilities we can converse with ELIZA like so:

Test:
```
HOW DO YOU DO. PLEASE TELL ME YOUR PROBLEM
> My mother hates me.
TELL ME MORE ABOUT YOUR FAMILY
> Derp.
DOES THAT HAVE ANYTHING TO DO WITH THE FACT THAT YOUR MOTHER HATES YOU?
```

Newkey/Equivalence
==================

Next, let’s take care of two other possible action types (or reassembly rules, as the paper calls them).

Our extended Action data type looks like this:

{% highlight haskell %}
data Action = Response List String
            | Equivalence String
            | Newkey
{% endhighlight %}

`Equivalence String` signifies that this keyword should be treated as another
given keyword.

`Newkey` means that this keyword should be ignored and another one of a lower
priority should be picked.

Both will be [handled](https://github.com/bartosz-witkowski/books/blob/master/natural-language-processsing-for-prolog-programmers/chapter-02/eliza.pl#L314-L320)
when searching for a matching script.

If `newkey` is the highest priority action the next script in the list is
considered.

If `equivalence(Other)` is the highest priority action - the script with the
keyword `Other` is appended into the list of considered scripts and the current
action counter of the keyword/pattern is bumped, the execution continues from
the new keyword.

A script which exhibits both new actions is the **remember** keyword.
Translating it and the **what** keyword into prolog:

{% highlight prolog %}
scripts(
  script(
    keyword(remember, 5),
    [
      pattern(
        matched([_, you, remember, X]),
        actions([
          response([do, you, often, think, of, X, ?]),
          response([does, thinking, of, X, bring, anything, else, to, mind, ?]),
          response([what, else, do, you, remember, ?]),
          response([why, do, you, remember, X, just, now, ?]),
          response([what, in, the, present, situation, reminds, you, of, X, ?]),
          response([what, is, the, connection, between, me, and, X, ?])
        ])
      ),
      pattern(
         matched([X, do, i, remember, Y]),
         actions([
           response([did, you, think, i, would, forget, Y, ?]),
           response([why, do, you, think, i, should, recall, Y, now, ?]),
           response([what, about, Y, ?]),
           equivalence(what),
           response([you, mentioned, Y, ?])
         ])
      ),
      pattern(
        matched([_]),
        actions([
          newkey
        ])
      )
    ]
  )
).

scripts(
  script(
    keyword(what, 0),
    [
      pattern(
        matched([_]),
        actions([
          response([why, do, you, ask, ?]),
          response([does, that, question, interest, you, ?]),
          response([what, is, it, you, really, want, to, know, ?]) ,
          response([are, such, questions, much, on, your, mind, ?]),
          response([what, answer, would, please, you, most, ?]),
          response([what, do, you, think, ?]),
          response([what, comes, to, your, mind, when, you, ask, that, ?]),
          response([have, you, asked, such, question, before, ?]),
          response([have, you, asked, anyone, else, ?])
        ])
      )
    ]
  )
).
{% endhighlight %}

Testing:

```
HOW DO YOU DO. PLEASE TELL ME YOUR PROBLE
> Do you remember the 21st night of September?
DID YOU THINK I WOULD FORGET THE 21ST NIGHT OF SEPTEMBER?
> Do you remember the 21st night of September?
WHY DO YOU THINK I SHOULD RECALL THE 21ST NIGHT OF SEPTEMBER NOW?
> Do you remember the 21st night of September?
WHAT ABOUT THE 21ST NIGHT OF SEPTEMBER?
> Do you remember the 21st night of September?
WHY DO YOU ASK?
> Do you remember the 21st night of September?
YOU MENTIONED THE 21ST NIGHT OF SEPTEMBER?
> Remember.
I AM NOT SURE I UNDERSTAND YOU FULLY
```

Replication Crisis
==================

Believe it or not this is all that is needed to implement ELIZA as descried in
the article. However, after implementing all the other scenarios I noticed
certain discrepancies between the behaviour of my implementation and sample
script in the article:

```
HOW DO YOU DO. PLEASE TELL ME YOUR PROBLEM
> Men are all alike
IN WHAT WAY?
> They're always bugging us about something or other.
CAN YOU THINK OF A SPECIFIC EXAMPLE?
> Well, my boyfriend made me come here.
YOUR BOYFRIEND MADE YOU COME HERE?
> He says I'm depressed much of the time.
I AM SORRY TO HEAR YOU ARE DEPRESSED
> It's true. I'm unhappy.
DO YOU THINK COMMING HERE WILL HELP YOU NOT BE UNHAPPY?
> I need some help, that much seems certain.
WHAT WOULD IT MEAN TO YOU IF YOU GOT SOME HELP?
> Perhaps I could learn to get along with my mother.
TELL ME MORE ABOUT YOUR FAMILY
> My mother takes care of me.
WHO ELSE IN YOUR FAMILY TAKES CARE OF YOU?
> My father.
YOUR FATHER?
> You are like my father in some ways.
WHAT RESEMBLANCE DO YOU SEE?
> You are not very aggressive but I think you don't want me to notice that.
WHAT MAKES YOU THINK I AM NOT VERY AGGRESSIVE BUT YOU THINK I DON'T WANT YOU TO NOTICE THAT?
```

It seems that `but` is another split word that triggers sentence splitting. The
[fix](https://github.com/bartosz-witkowski/books/blob/master/natural-language-processsing-for-prolog-programmers/chapter-02/eliza.pl#L98-L123)
was simple enough thankfully.

Another discrepancy was not far away however.
```
    [...]
WHAT RESEMBLANCE DO YOU SEE?
> You are not very aggressive but I think you don't want me to notice that.
YOU SAY I DON'T ARGUE WITH YOU
```

which differs from the response “WHY DO YOU THINK I DON’T ARGUE WITH YOU?”. In
this case it turns out that **you** and **i** have the same priority. It’s only
a matter of luck that in the original paper **i’m** was preferred to **you**.

In this case I’ve decided to complicate matters a bit. In my mind the
`matched([X])` pattern from the **you** script is less general than the
`matched([_, i, remind, you, of, _])` pattern from the i keyword. So I decided
to add
[scoring](https://github.com/bartosz-witkowski/books/blob/master/natural-language-processsing-for-prolog-programmers/chapter-02/eliza.pl#L643-L657)
to the patterns and resolve ties on keywords with how 
[well patterns match](https://github.com/bartosz-witkowski/books/blob/master/natural-language-processsing-for-prolog-programmers/chapter-02/eliza.pl#L251-L332).

Afterthoughts
=============

I checked out some reference implementations of ELIZA before delving into
writing my own, I didn’t like them much, yet as far as my own code goes it’s not
exactly clean (though I think it’s mostly due to the sheer amount of debugging
added). The two I found didn’t manage to recreate the example script from the
article.

This is really my first prolog that’s longer than a few screens and I think it
shows. I really missed strong typing guarantees, and even though sometimes the
dynamic nature of prolog was useful (e.g adding word classes). I guess my next
project should be adding some less ad-hoc type checking to prolog.
