---
layout: post
title: Creating classes in LaTeX.
permalink: /2013/07/15/creating-classes-in-latex.html
categories:
- blog
---

I've decided to revamp my resume. The code of the previous one starting to be an
awful mess. Because LaTeX classes/styles were always magic to me I've decided to
implement my own to use as a base for my resume.

This post will cover some basic latex macros and how to write your own class.

Debugging and diagnostics.
==========================

I've found that latex by default is very quiet and sometimes when an error
occurs it doesn't show enough to know what is actually wrong 

The `errorcontextlines` lines macro is useful to diagnose what's really going on
when latex burps up an error. 

Apart from that sometimes it's useful to trace macros (or other things), I don't
have this turned on by default when I compile a document but sometimes it's
needed. For more on tracing see: [http://tex.stackexchange.com/questions/60491/latex-tracing-commands-list](http://tex.stackexchange.com/questions/60491/latex-tracing-commands-list)

Setting the `draft` option on in the article class will visually show overfull
boxes and it helps tweaking so much!

Writing your own classes.
=========================

Is actually pretty easy. Here are some links that may, or may not be useful for
someone trying to create your own class:

* [http://en.wikibooks.org/wiki/LaTeX/Creating_Packages](http://en.wikibooks.org/wiki/LaTeX/Creating_Packages) 
* [Minutes in Less Than Hours: Using LaTeX * Resources](http://tutex.tug.org/pracjourn/2005-4/hefferon/)

This part of the article will be mostly a rehash of those, with a little
explanation and notes of my own.

Before we start diving deep into latex I've created an minimal file that uses the style:

{% highlight latex %}
\errorcontextlines 10000
\documentclass[a4paper,20pt]{cv}
\usepackage[T1]{fontenc}	
\usepackage[utf8]{inputenc}	

\begin{document}
Foo
\end{document}
{% endhighlight %}


There are of course some magical incantations but a minimal style file is
surprisingly short:

{% highlight latex %}
\ProvidesClass{cv}[2013/07/14]

\DeclareOption*{\PassOptionsToClass{\CurrentOption}{article}}

\ProcessOptions \relax

\LoadClass{article}
{% endhighlight %}

The `\ProvidesClass{...}` macro declares the actual class named in my case `cv`.
It needs to be in a file called `CLASS_NAME.cls` - in my case `cv.cls`. The
optional argument in the square brackets denotes the version (so that the
package users can specify a minimal class version).

The next macro `\DeclareOption*...` passes the options given to this class to
the article class that we use as a base.

`\ProcessOptions \relax` finishes processing options so that we can load the
class. As [this post](http://www.latex-community.org/forum/viewtopic.php?f=5&p=69060) says the
`\relax` option does nothing (kind of like an nop) but safely stops the
expansion of the `ProcessOptions` macro.

Finally the `\LoadClass` macro loads the article class.

Class innards
=============

Having that out of the way let's actually do something useful. The rest of the
document will be more about wiring macros and using latex with it's little
idiosyncrasies.

Setup
-----

The first thing I wanted to add is a simple way of defining a resume header,
setting up pdf properties and so on. On the use site it should look as follows:

{% highlight latex %}
\fullname{Bartosz Witkowski}
% ...
\begin{document}
\mkheader
% ...
{% endhighlight %}

And in effect it prints out "Curriculum Vitae" and my full name, and sets up the
pdf properties.

When writing macros I usually write "output first", so first I would write:

{% highlight latex %}
{\centering{\Large{\textsc{Curriculum Vitae}}} \\}
{\centering{\LARGE{\textsc{Bartosz Witkowski}}} \\}
\hypersetup{
	pdftitle={Bartosz Witkowski - Curriculum Vitae},
	pdfauthor={Bartosz Witkowski}
}
{% endhighlight %}

The `\hypersetup` macro requires the `hyperref` package and I'll add it straight
to the class file (` \RequirePackage{hyperref}`)

On the second stage I would set up the "variable".

{% highlight latex %}
\def \@fullname {none}
\newcommand {\fullname}[1]{\def \@fullname{#1}}
{% endhighlight %}

The second line may be thought of as a setter. The `@` symbol in latex is kind
of special and cannot be used in user code without some tricks (surrounding it's
usage int in `makeatletter` and `makeatother`). But in classes we can use it as
we like.

After having that defined I modify my output to form a macro:

{% highlight latex %}
\newcommand{\mkheader}{
   {\centering{\Large{\textsc{Curriculum Vitae}}} \\}
   {\centering{\LARGE{\textsc{\@fullname}}} \\}
   \hypersetup{
     pdftitle={\@fullname - Curriculum Vitae},
     pdfauthor={\@fullname}
   }
}
{% endhighlight %}

Another thing I wanted to get out of the way from my cv was margins so I've
added this to the class file:

{% highlight latex %}
\RequirePackage[
   top   = 3cm, 
   bottom= 2.5cm, 
   left=3cm, 
   right=2cm]{geometry}
{% endhighlight %}

WYSIWYM
-------

The biggest problem with my previous cv file was that I've lost the WYSIWYM
property. The file was a big clunk of tables, `\par`s `\vskip`s etc. The new
file shouldn't care about all that jazz - just meaning.

To highlight what I mean I've modified the sample file and added two sections.

{% highlight latex %}

\section{Personal Information}

\begin{description}
   \item[Date of Birth] { 1970-01-01 }
   \item[Address]       { ul. Wiejska 4/6, 00--902 Warsaw, Poland }
   \item[Phone number]  { (+48)012 345 678 }
   \item[E-mail]        { john-smith@example.org }
\end{description}

%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%

\section{Work Experience}

\begin{description}
  \item[01.2012 -- present]{
     \begin{itemize}
        \item {
           Intercal programmer at Acme, Warsaw. 
        }
        \item {
           Responsible for foo, bar, baz and qux.
        }
        \item {
           {\bf Keywoards:} Intercall, Make, etc
        }
     \end{itemize}
   }
\item[01.2011 -- 12.2011]{
   \begin{itemize}
      \item {
         Lolcode Programmer -- Lolcats Ltd, Gotham. 
      }
      \item {
         I CAN HAZ CHEESBURGER
      }
      \item {
         How did I get here I'm not good with computer
      }
      \item {
         {\bf Keywoards:} Lolcode.
      }
   \end{itemize}
}
\end{description}
{% endhighlight %}

And this is about 90% of my use case from the resume. So what I needed to do is
to modify `itemize` and `description` environments to do exactly what I want.

I've opted to redefine them completely like so:

{% highlight latex %}
\renewenvironment{itemize}
   { 
      \let \@@olditem \item
      \renewcommand{\item}[2][]{
         {##1} {##2}  \\ \noalign{\vskip 1mm}     
      }
      \noindent
      \begin{tabular}[t]{@{}p{0.95 \linewidth}@{}}
   } {
      \end{tabular}
      \renewcommand{\item}[2][]{
         \@@olditem[##1]{##2}
      }
   }


\renewenvironment{description}
   {
      \let \@olditem \item
      \renewcommand{\item}[2][]{
      \begin{hyphenrules}{nohyphenation}
        \textsc{##1}
      \end{hyphenrules}
         & ##2 \\ 
      }
      \begin{tabular}{@{}p{0.25 \linewidth} p{0.65\linewidth}@{}}
   } {
      \end{tabular}
   \  \renewcommand{\item}[2][]{
         \@olditem[##1]{##2}
      }
   }
{% endhighlight %}

The `let @olditem item` and `renewcommand item @olditem` commands let us
redefine the item macro only in a localized context so that if we wish to use it
in another place we could. 

You may notice that the parameters for `item` macros come with two hashes like:
`##1`. When we define nested macros we need to add another hash for every level.
This really tripped me at the beginning, as I did not know this.

I'll start from the `description` environment - I've decided to reuse the tabular
environment. The `\item` macro will insert a description in the first row
written in small capitals (`\textsc`) along with the item content in the second
row. The `nohyphenation` comes from observation - hyphenating words in such a
small row looks really wrong to my eyes. The paragraph widths in the table were
adjusted experimentally until "it looked right".

The `itemize` environment was redefined to insert a small skip between every
item. Apart from that it's really vanilla. We had to redefine it because
normally `itemize` and tabular environments don't really mix so well.

Footer.
=======

Lastly, I've defined a little footer macro. In Poland, at least, resumes to be
used in a recruitment process have to have a little legal disclaimer.

{% highlight latex %}
\newcommand{\footer}[1]{
\null \vfill
\begin{center}
   \textsf { 
      \textcolor[rgb]{0.3,0.3,0.3}{
         \scriptsize #1
      }
   }
\end{center}
}
{% endhighlight %}

This will flush the page and insert the centered footer on the bottom.

That's it! I've published my resume class on github, along with a small sample.
You can grab it [here](https://github.com/bartosz-witkowski/latex-cv-class). Or
you can backtrack to the [main page](/) and see my resume for yourself.
