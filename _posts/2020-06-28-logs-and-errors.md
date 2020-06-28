---
title: "To Log or not to Log?"
date: 2020-06-28
categories:
  - blog
tags:
  - logs
  - golang
---

Logs and me have a long time relationship. I have been depending on them
from my early days as an integration and support engineer up to the current
times where I focus on development and coding.
There were times when they saved me from painful debugging/troubleshooting
and there were times when I felt I am drowning in a mud of meaningless
information, helpless in my attempts to distinguishing between the useful
information and the noise. 

Logs have a critical role in a system, providing valuable information about
the system behavior and state. Its real usefulness is exposed when something
unexpected happens and everyone starts digging into them to debug and
troubleshot the system.

We love them when they are clear, easy to read and provide the needed info
to find the problem. Unfortunately, it is hard to balance between clarity
and the amount of information they provide. Too little information is not
helpful and too much information is overloading and confusing.
Balancing between these is hard, but we can try and manage it with some
rules.

## Application Errors

Errors may cause an application to either stop working (panic, as it has no
way to recover from the error) or to recover fully or partially. In the later
case, the error is usually recorded on the log for later analysis.

An error may be recorded in the log with limited information which describes
what occurred (e.g. "file foo.txt cannot be opened"). On the other hand,
a full stack-trace may be dumped, describing the path in the code that was
exercised to reach the error. 

## Balance

Naturally, developers are driven to push for more logging to help with
the debugging while support and advance users who peek into them will prefer
readability and a more abstract approach. The later group will rarely invest
in following the code-base to understand the logs, making some log entries
both irrelevant and harmful.

Log [severity levels](https://tools.ietf.org/html/rfc5424#section-6.2.1)
have been established in order to serve both needs of the previous two groups.
At production, only high level informative logs are reported while during
testing or a debugging session additional logs are exposed.

Even an attempt to define the level of details a developer needs to debug the
application is debatable.
Should the application have a backtrace dump on every error encountered in the
application? Will it always help or will the overloaded information make it
harder to debug the overall system.
There is no hard-rule here, it depends on the application and the number of
errors which it may encounter during its operation.

## Debugging with logs

Identifying when a log entry is needed for an application user or a developer
that needs to debugs its operation is a relative easy task.

Identifying when a log entry may be useful for debugging to other developers
that will maintain the code (in the future) is much harder.

It is trivial to just log everything and debug the behavior during development,
but passing these logs on to tests and your fellow developers may not be always
appropriate.

There is a need to think carefully about the logging pattern used in a project.
Keeping the logs in good shape, clear, focus and supporting change.

## Logs are not free

With all the goodies there is always a cost, logs are not free.
The amount of logs, the information they share and the resource cost are
factors to be considered when asking "to log or not to log?".

Lets list some of the cost logs carry with them:
- Resources: Logs, especially a large amount of them may have CPU, memory and
  storage implications.
  This may hit distributed systems which are build to scale.
- Obscurity: Too much logging entries may make it difficult to identify the
  flow of operations or hide the important details in the forest of low level
  details.
- Inaccurate: It is hard and sometimes wrong to debug concurrent behavior
  through logs (be it threads, processes or simple go-routines). Logs are
  not atomic and time signatures are not assured to correspond to the actual
  execution of the log or its surrounding operation.
- Context: Information like backtraces may sometime lack context (e.g. the
  arguments each function was called with) and therefore is less useful as
  a generic data dump.
- Other side effects: In golang, one known side effect of a log or print
  (depending on the implementation) is the io operation which may trigger
  a go-routine rescheduling.

## Telemetries

On high performance application (e.g. packet processing) logging is impractical
and telemetries are used to collect information for later analysis.

Such patterns may be used to collect information and analyse suspected areas
of the code without overloading the logs with endless entries which are hard
to analyse.
This is especially useful in concurrent operations where the logs are not
useful in identifying the sequence of events.

## Double logging

Double logging occurs when both the caller and one of its callee down the stack
provide the same information but through two or more log entries.

This may be a side effect of logging in helper functions and not in the main
business logic flow.
See [helper functions and libraries](#helper-functions-and-libraries)
for more details.

Here is a simple example which illustrates double logging:
```golang
func caller1() {
    if err := helper(); err != nil {
        log.Printf("helper failed with %v", err)
    }
}

func caller2() {
    if err := helper(); err != nil {
        log.Printf("helper failed with %v", err)
    }
}

func helper() error {
    if err := foo(); err != nil {
        log.Printf("foo failed with %v", err)
        return err
    }
}
```

## Helper Functions and Libraries

Would you expect any library you import to log?
I would argue that in most cases the answer is **no**.

Libraries and helpers are in place to serve callers, they usually have little
context on when and why they have been called and they cannot assume how many
times or the rate they are going to be called.

Logging in helpers and libraries at runtime may result in logging the calling
path multiple times. Even with a single path logging, the developer is left to
walk through the calling stack and guess who call it. In both cases, they
become a no-help and pure noise.

> But wait, what about collecting data that is available only deep inside the
calling stack?

- If appropriate, pop up the information with an error.
- Consider separating data collection and the operation on it. e.g. `do(get())`
- Consider appending trace metadata on objects (pattern used with high packet
  processing implementations).
  When appropriate, dump the trace information.
- There are always exceptions, log if you must.

> Where do you expect me to log then?

Prefer logging at the business logic flow and not deep down the calling stack.
As deeper in the stack you are, the less appropriate it is to log.

## Error Entities

Most programming languages treat errors either by raising exceptions or by
returning them back to the caller.
Golang decided to use the later and requires explicit treatment of errors.

An error in Go has 3 stages:
- Detecting a condition that merits an error, creating the error and
  returning it to the caller.
- Forwarding a (returned) error.
- Handling a (returned) error.

When the error is created and returned to the caller, all relevant information
may be included as its value, therefore, logging is redundant. The caller is
left to decide if it will be useful to log the error and its content.

When an error is forwarded, one may still need to collect data for
troubleshooting. Instead of logging, prefer appending the relevant data on
the error.

A simple example may look like this:
```golang
if err != nil {
    return fmt.Errorf("processing %v: %v", name, err)
}
```

Finally, when an error is handled, log it.

Due to the frequent usage of error stacking, go 1.13 introduced
[new features](https://blog.golang.org/go1.13-errors) to the
[errors](https://golang.org/pkg/errors/) package.
These enhancements allows the exposure of errors as API/s in an easy-to-use
manner.

## Severities

What log severities should be used and when, is a topic by itself.
It is enough to mention here that as little levels we use, the better. It makes
the decision simpler for the developer and much clearer for the log reader
on how to treat it.

This has been a subject of
[discussion](https://dave.cheney.net/2015/11/05/lets-talk-about-logging)
with golang logging in particular.

Debug severity is many times reasoned for when logging a questionable part of
the code. However, populating the code with logs at inappropriate locations
may still present the same drawback discussed earlier, depending on the usage
circumstances (debug severity enabled during e2e testing, initial releases).

## Reason for exceptions

There are always exceptions to any pattern or rule.
If you recognize it is an exception, reason for it.

## Conclusion

Dependent on the system, balance the logging in a way it will be useful and
not a burden.
Consider both logs consumers and its effective usefulness when appearing in
code.

Logs used during development which have been helpful to the developer are not
a good enough reason to keep them in. They need to make sense when the system
is deployed in the field.
