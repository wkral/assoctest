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
    - This can be done piecemeal a file at a time as they're saved
        - There probably has to be an initialization parsing for a new project
    - Ideally there can be a file with the saved parsing information that can be checked into source control
        - Even better if it could be self healing from any merge conflicts
- All the code needs to be classified as testing code vs program code
    - A solved problem using some other test frameworks code here might be useful
- A data structure with an association strength value between test code and program code
    - Needs to be based on some level of history, more recent events should affect the strength than past events
    - Should be saved and sharable in source control
        - Needs to be able to be merged
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
