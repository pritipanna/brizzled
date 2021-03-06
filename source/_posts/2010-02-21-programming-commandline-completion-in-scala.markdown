---
layout: post
comments: true
title: "Programming Command-line Completion in Scala"
date: 2010-02-21 00:00
categories: [scala, parsing, readline, editline, REPL, completion, programming]
---

I wrote a [Scala][]-based SQL [REPL][] called [SQLShell][]. To make it more
portable, I chose to write a Scala readline wrapper API that talks to
multiple underlying readline implementations; that way, SQLShell could use
whatever was available, without the need for special-case code. (The
readline wrapper API, which supports [editline][], [GNU Readline][] and
[JLine][], is available in my [Grizzled Scala][] library. See the
`grizzled.cmd` and `grizzled.readline` packages.)

Naturally, I wanted to support tab completion. But, as it happens, most
completion APIs are a little clunky. They give a bare minimum of
information, leaving a fair amount of work to the caller.

<!-- more -->

For example, the [Python][] `readline` module provides for tab completion;
the completion function, according to the module's documentation, "is
called as *function(text, state)*, for *state* in 0, 1, 2, ..., until it
returns a non-string value. It should return the next possible completion
starting with *text*."

Well, *that's* ugly. But, in all fairness, it merely mimics the ugly
approach used by the underlying GNU Readline API. GNU Readline itself is
considerably [more complicated][].

Another example is [editline][]. You can install a completion callback,
which receives the Editline descriptor and a character. You can then query
the API for the information about the current line; you get back a
`LineInfo` structure that looks like this:

{% codeblock lang:c %}
    typedef struct lineinfo
    {
      const char *buffer;
      const char *cursor;
      const char *lastchar;
    }
    LineInfo;
{% endcodeblock %}

From that structure, you know three things:

- The contents of the current buffer (which can contain more than
  the current line).
- The last character in the buffer (i.e., the end of the current line).
- The location of the cursor in the buffer.

You have to write your own code to find the token being completed.

These approaches have a couple problems.

First, every client program tends to do the same thing. Every
Editline program, for instance, contains similar code to find the
token being completed.

Second, the typical completion handler's code isn't exactly
straightforward and easy to read. By necessity, it mixes lexical
parsing (e.g., to find the token) with semantic interpretation
(e.g., What does this token mean if it's here in a line, as opposed
to there?)

Using Scala pattern matching, I was able to craft a solution that
allows my client code completion handlers to focus primarily on the
semantics. You might find this completion approach interesting, or
you may find it appalling. I wasn't sure myself whether I was happy
with it, until recently, when I had to fix a completion bug. I
found that this approach made it very clear what was going on in
the completion handler, and the bug was trivial to fix.

The easiest way to describe the approach is to show how it handles
the `.desc` command in SQLShell. `.desc` is used to describe
several things:

`.desc database` describes the currently connected database. For
example:

{% codeblock lang:bash %}
    sqlshell> .desc database
    Connected to database: jdbc:mysql://allegro:3306/bmc
    Connected as user:     bmc@localhost
    Database vendor:       MySQL
    Database version:      5.1.37-1ubuntu5.1
    JDBC driver:           MySQL-AB JDBC Driver
    JDBC driver version:   mysql-connector-java-5.1.7 ( Revision: ${svn.Revision} )
    Transaction isolation: repeatable read
    Open transaction?      no
{% endcodeblock %}

`.desc table` is used to describe a table. For example:

{% codeblock lang:bash %}
    sqlshell> .desc names
    -----------
    Table names
    -----------
    id             int4 NOT NULL,
    firstname      varchar(20) NOT NULL,
    lastname       varchar(20) NOT NULL,
    gender         bpchar(1) NOT NULL,
    middleinitial  bpchar(1) NULL,
    creationdate   timestamp NOT NULL
{% endcodeblock %}

With the addition of the string "full", it also gets index
information:

{% codeblock lang:bash %}
    sqlshell> .desc names full
    -----------
    Table names
    -----------
    id             int4 NOT NULL,
    firstname      varchar(20) NOT NULL,
    lastname       varchar(20) NOT NULL,
    gender         bpchar(1) NOT NULL,
    middleinitial  bpchar(1) NULL,
    creationdate   timestamp NOT NULL
    
    Primary key columns: id
    
    names_pkey: Unique index on (id)
    namesfirstix: Non-unique index on (firstname)
    nameslastix: Non-unique index on (lastname)
{% endcodeblock %}

Thus, the basic command forms are:

{% codeblock lang:bash %}
    .desc database
    .desc table [full]
{% endcodeblock %}

Here are some of the completion challenges. In the examples, below,
the location of the cursor is indicated with \[\].

A tab pressed at the location of the cursor, below, should complete
".desc", since there are no other commands starting with ".d":

{% codeblock lang:bash %}
    sqlshell> .d[]
{% endcodeblock %}

By contrast, a tab pressed here does nothing, because the command
is already completed:

{% codeblock lang:bash %}
    sqlshell> .desc[]
{% endcodeblock %}

In this next case, a tab pressed here should show the choices
"database" and the list of tables that are available for
completion:

{% codeblock lang:bash %}
    sqlshell> .desc []
    database
    foo
    names
{% endcodeblock %}

Here, a tab should complete "foo", since there's a table named
"foo", and no other candidate starting with "f":

{% codeblock lang:bash %}
    sqlshell> .desc f[]
{% endcodeblock %}

In both of the following cases, pressing a tab should complete the
word "full":

{% codeblock lang:bash %}
    sqlshell> .desc foo []
    sqlshell> .desc foo f[]
{% endcodeblock %}

To make this kind of parsing easier to model, my Scala readline
adapter API converts the line into a list of tokens.

* A text token is stored in a `Some` object.
* White space (the delimiter) is represented by `Delim` objects.
  All adjacent white space is collapsed into a single delimiter.
* The cursor is represented by a special `Cursor` token.
* The end of the token stream is denoted by `Nil`.

Given this input:

{% codeblock lang:bash %}
    sqlshell> .desc foo f[]
{% endcodeblock %}

the API produces this token list:

{% codeblock lang:scala %}
    Some(".desc") :: Delim :: Some("foo") :: Delim :: Some("f") :: Cursor :: Nil
{% endcodeblock %}

Similarly, this input:

{% codeblock lang:bash %}
    sqlshell> .desc []
{% endcodeblock %}

produces this token list:

{% codeblock lang:scala %}
    Some(".desc") :: Delim :: Cursor :: Nil
{% endcodeblock %}

With that approach, writing a completion handler is pretty
straightforward:

{% codeblock lang:scala %}
    override def complete(token: String, allTokens: List[CompletionToken], line: String): List[String] = {
      allTokens match {
        case Nil =>
          // Should not be called on an empty line.
          Nil
    
        case LineToken(cmd) :: Delim :: Cursor :: Nil =>
          // Command filled in (obviously, or we wouldn't be in here),
          // but first argument not. subCommandCompleter completes
          // for subcommands.
          subCommandCompleter.complete(token, allTokens, line)
    
        case LineToken(cmd) :: Delim :: LineToken("database") :: Cursor :: Nil =>
          // Nothing more after ".desc database"
          Nil
    
        case LineToken(cmd) :: Delim :: LineToken(table) :: Delim :: Cursor :: Nil =>
          // Cursor is after a table name (and white space). Only
          // "full" can be completed here.
          List("full")
    
        case LineToken(cmd) :: Delim :: LineToken(table) :: Delim :: LineToken(arg) :: Cursor :: Nil =>
          // If it can't complete "full", return nothing.
          if ("full".startsWith(arg))
            List("full")
          else
            Nil
    
        case LineToken(cmd) :: Delim :: LineToken(arg) :: Cursor :: Nil =>
          subCommandCompleter.complete(token, allTokens, line)
    
        case _ =>
          Nil
      }
    }
{% endcodeblock %}

(The `subCommandCompleter` object is an instance of a stock "List"
completer that is instantiated with a list of choices ("database" and the
list of tables, in this case) and returns zero, one or many completions
from that list. Completing from a list of choices is common, so the API
provides an easy way to do that.)

This matching-based approach hides the nitty gritty parsing
details, allowing the completer to focus on the "business logic" of
figuring out context and returning the appropriate token. For me,
it also more closely mimics how I mentally model the command line
being completed.

[Scala]: http://www.scala-lang.org/
[REPL]: http://en.wikipedia.org/wiki/Read-eval-print_loop
[SQLShell]: http://software.clapper.org/scala/sqlshell/
[editline]: http://www.thrysoee.dk/editline/
[GNU Readline]: http://tiswww.case.edu/php/chet/readline/rltop.html
[JLine]: http://jline.sourceforge.net/
[Grizzled Scala]: http://software.clapper.org/scala/grizzled-scala/
[Python]: http://www.python.org/
[more complicated]: http://tiswww.case.edu/php/chet/readline/readline.html#SEC44
[editline]: http://www.thrysoee.dk/editline/
