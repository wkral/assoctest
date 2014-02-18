Concept
-------

A test runner that builds associations between test and project code by observing file changes and test failures.

Features
--------

- The tests most closely associated with the code that was changed are run first
- Failing tests from previous passes are run before passing tests in association order
- If the change is in a test itself that is run first
- New code changes will interrupt a test suite running
- Shared setup resources can be initialized at startup of the test runner as long as they can be reset effectively
- If the changed code results in a syntax error the tests shouldn't be run


Implementation Thoughts
-----------------------

- The entire code base of the project will need to be parsed and descretized into smaller parts
    - Functions and Methods seem like the right size
    - Use the languages own AST parser
        - MR: Initial thought is we can do this quick'n'dirty w/ regexes to detect function/method begin/end
        - WK: Figured the parsers are there for most languages so that would be quick but not opposed to quick regex
    - This can be done piecemeal a file at a time as they're saved
        - There probably has to be an initialization parsing for a new project
    - Ideally there can be a file with the saved parsing information that can be checked into source control
        - Even better if it could be self healing from any merge conflicts
        - MR: Not sure that its a good idea to check in a parsed form of the project code. Parsing should be cheap enough that it can be done as needed.
        - WK: Based on recent experiences I tend to agree, but it should at least be incremental
- All the code needs to be classified as testing code vs program code
    - A solved problem using some other test frameworks code here might be useful
    - MR: Also it is critical that we can map test output (failures) to an addressable testcase
    - WK: Pretty much all existing test frameworks do this so shouldn't be a problem
- A data structure with an association strength value between test code and program code
    - Needs to be based on some level of history, more recent events should affect the strength than past events
    - Should be saved and sharable in source control
        - Needs to be able to be merged
        - WK: Just want to make the case for getting this right since if it's mergable data from various people working on different machines could effect the associations
    - Could be an append only list of historical events that is computed into association scores at startup
        - A sequence of files of a particular size
        - Once a file has events too far in the past it can be deleted
        - Maybe each user has separare set of files all shared together so there is no conflicts
- File changes can be reported by the users editor to a listener which parses that file
- There needs to be some level of process or thread hierarchy
    - Master process that handles the parsing and metadata
    - Interruptable workers which run the tests and report back
- Internal state of which tests are currently failing
    - Can be gathered from running the suite on startup
    - No need to persist
- A library provided to the user to assist in writing tests
    - User can mark some setup code as global which the master process can initialize other setup run by workers
    - Shared setup between groups of tests could be a problem as they're not run in the same order
        - Eg. Add some records to the database that the next 15 tests will all use
        - Might have to setup and teardown multiple times

Mike's Thoughts
--------------

2014-01-16:

My biggest concern is tackling this in a language agnostic way. In an ideal world we can work
on this along-side our other work. For me that means Lua.

As an implementation lang though I'm open to thoughts. Python is probably a good choice. I suspect
it would be your first choice.

Do you know any algorithms that are immediately applicable to solving the association score? Seems
like its a pretty basic machine learning algo.

To start I wonder if its easiest to write the 'daemon' as something that just reads changed file
names from stdin, and then launches the tests based on those events.

This works well on Linux because there are command line tools that will watch a directory of files
and dump the change events to stdout. Not sure of OS X has anything equivalent, but it'd be a
convenient way to defer solving the file-monitoring problem.

Will's Reply
------------

2014-02-17:

I think it would be worth it to implement this in one language first without
having to worry about making it agnostic and solving that problem later after
we get the first one to work. I'm happy to learn Lua and use that. Might even
be good as the main language down the line as I understand it's fairly light
weight and embeddable.

I don't know any strong candidates but I sort of look at it like a probability
calculation. It would roughly be stored as a matrix of tests and program units
with the probability that the program unit effects that test at the coordinates
T<sub>i</sub> P<sub>j</sub>

The naive way would just be to count up all the events and divide each count by
the total number of events. There is likely some way of doing that
incrementally. I want to say Baysian Probability applies here somehow. But
needs more research.

I've been through the gamut on file change monitoring and here are my thougts:

- Simple polling checksum every few seconds
    - eats CPU especially in a VM
    - high latency
- epoll/kqueues (linux/bsd osx file monitors)
    - Good for CPU
    - Very Responsive
    - Eats file handles
    - Can sometimes have trouble with newly created files
    - Or recursive monitoring of subdirectories
    - VitualBox shared filesystem doesn't fire filesystem events
        - I use vagrant/vitrualbox for dev most of the time now
- Server listener
    - Good for CPU
    - Very Responsive
    - Have to implement plugin for every editor
    - Can't do it for some editors

So no ideal solutions here. I'm currently using the server listener approach as
it was fairly trivial to implement the plugin for vim, but it's probably not a
long term solution. I'd love to have a proper file system event listener but
I've always found there is something wrong with them. But I'm open to seeing if
some work. They also don't seem to scale when there are lots of files in a
project. But maybe decoupling the runner from the method of finding changes is
a good idea either way.

So recently I've been writing ruby code at work and I tried using something
called guard. But it seems to not work so well. It's approach is to write a
bunch of regex and globs so when certain files change certain things are
restarted to support the test infrastructure. For example in rails changing
some files requires a test server restart.

I'd like to handle it down the line rather than up front but it seems like it
might be a problem that needs solving. That is the shared resource problem.
Some heavier components like databases may be required for tests. RSpec from
ruby actually has a pretty decent way of denoting shared setup code by nesting
blocks of setup code but that does impose a certain order on things.

For some shared resources I think they can almost be thought of like system
services with multiple levels of resets like hard restart and soft reload. I
think it wouldn't be a very long shot for the system to automatically figure
out which resources are needed by which tests instead of having to declare that
in the test code everywhere. That could be done using the same trial and error
mechanism.

A version 2 type bonus could be generators that could maybe spin up 4-5
databases to run some tests in parallel with a maximum for each resource and
the test runner uses a lock on the resource to introduce some serialization.
