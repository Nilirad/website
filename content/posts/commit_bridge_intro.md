+++
title = "What I learned about backend development from a greenfield project"
date = 2026-07-04
+++

I spent the last three months creating
a server that dispatches workflows for git dependencies.
The server periodically monitors branches on remote repositories,
and when it detects a change,
it sends an event to the subscribed GitHub repositories,
triggering a workflow.
This allows the target repository to run a series of automated checks
to ensure that it is still compatible with the upstream repository.

I'll be the first to admit this is quite a niche idea,
so a question naturally arises: where did it come from?
Last year, I started participating in a [Bevy] working group
to prototype an entity inspector for running Bevy apps.
The Bevy community has a strong dogfooding philosophy;
therefore, the inspector was designed as a Bevy app
(i.e., using the `bevy` library crate as a dependency).
The development of this project happened outside of the Bevy repository,
so synchronizing with the fast-paced development of the engine
became a non-trivial problem.
We first depended on released versions,
but that meant missing new features by months,
and having to fix possibly obscure breaking changes after an upgrade.
Later, it was decided to track the `main` branch,
occasionally updating the lockfile to synchronize with the latest changes.

The concept of automating the synchronization of projects
that span across multiple repositories has fascinated me,
so I decided to create this solution as part of my portfolio.
While this solution might seem a bit overkill
(a workflow that updates the lockfile on a parallel branch every night
is enough for a single dependency),
it might also have its uses in more complex cases,
and building a project from the ground up
has truly been a precious learning experience.

[Bevy]: https://bevy.org/

## Development challenges

{% mermaid() %}
flowchart TD
    User[fa:fa-user User]
    User ----> API
    subgraph CommitBridge
        API[🧵 REST API] ---> subscriptions
        subscriptions(fa:fa-database subscriptions)
        branches(fa:fa-database branches)
        trigger_queue(fa:fa-database trigger queue)
        TriggerEngine[🧵 Trigger Engine]
        PollingEngine[🧵 Polling Engine]
        subscriptions -->|upsert| branches
        subscriptions --> TriggerEngine
        branches --> PollingEngine
        PollingEngine --> trigger_queue
        trigger_queue --> TriggerEngine
    end
    
    TriggerEngine ----> GitHubApi[GitHub API]
    PollingEngine ---->|git ls-remote| GitRepositories[Git Repositories]
{% end %}

The idea is simple:
the server monitors some branches,
and when an update is available,
it sends the event to the subscribed repositories.
However, like all engineering projects,
it presented many hidden complexities.
How is data going to be stored?
How is GitHub authentication handled?
What strategy is the best for tracking upstream branches for changes?
These and other questions
turned an apparently simple project into a challenging quest.

### Asynchronous tasks

Unlike any type of programming I have done in the past,
backend development is I/O-heavy,
meaning a server can have thousands of connections at any given time,
and satisfying each request takes a significant amount of time,
most of which is spent not processing data, but transferring it.
The traditional synchronous paradigm of fetching or sending data,
waiting for the operation to complete,
and then resuming the control flow
would result in a very tight performance bottleneck.

Asynchronous programming is supposed to fix this problem.
I admit that wrapping my head around this paradigm has been quite difficult
until I finally decided to build a real project that makes use of it.
My biggest conceptual struggle was the execution of the control flow.
It clicked when I examined the concept of cooperative multitasking,
which is a way cheaper alternative
to the operating system's preemptive multitasking.
While creating and executing an operating system thread is a costly operation,
in asynchronous programming there is the concept of _green threads_,
user-space threads managed by a runtime.
Green threads have a number of advantages over the native OS threads:
they have a smaller memory footprint and smaller context-switching costs,
which makes them much more scalable compared to their native counterparts.
Since it is the runtime that owns the control flow,
tracking asynchronous code
is not as clear as when you are reading synchronous code.
In Rust, when you encounter an `await` keyword after a function call,
you should think of it as the caller
_yielding_ the control flow of the native thread to the asynchronous runtime
([Tokio] in this case),
which allocates the CPU time to other tasks
while the awaited future resolves.

I used Tokio's tasks to apply the asynchronous paradigm in my project.
**Tasks** are the implementation of green threads
that Tokio schedules across a pool of native threads.
My application spawns three asynchronous tasks
that stay open for the entire process lifecycle:
an Axum HTTP server to manage subscriptions,
and two "engines" powering the business logic.
Each engine retrieves data from a source,
reacts to it,
and sleeps until it is woken up again,
or the server needs to be shut down.
The first one is the **polling engine**,
scanning monitored remote branches for updates,
while the other one is the **trigger engine**,
translating updates to `repository_dispatch` events for subscribers.

<!-- LINKS -->
[Tokio]: https://tokio.rs/

### The polling engine

One of the main questions I needed to answer before starting this project was:
how do I notify the server of changes in a repository?
Two possibilities came to mind:
Either the upstream repository workflow branch
sends a wake-up signal to the server each time a commit is pushed,
or the server periodically polls each repository for changes.
I opted for the latter,
even if there is a small delay between update and polling,
since it is simpler and does not require upstream collaboration.

Once that question was solved,
another arose:
how do I retrieve the last commit on a remote branch?
First, I learned about each GitHub repository having an Atom feed;
however, I quickly realized that downloading an entire XML file
that needs to be parsed to just extract a hash is too much work,
so I started looking for alternatives.
I finally landed on the `git ls-remote` command,
which returns the latest commit's hash
with some other information that can be easily stripped out.
Ironically, during a final review of this draft,
I realized this approach actually undermines
the asynchronous programming philosophy,
since launching the `git` command spawns a new process.
Anyway, the received hash is compared to the hash stored in the database,
and if there is a change, it is queued into an MPSC channel
to be sent to the trigger engine.

### The trigger engine

The trigger engine wakes up
each time it receives a message from the MPSC channel.
The main purpose of this engine
is to manage the contrived authentication process
to send the crucial `repository_dispatch` event needed to trigger the workflow.

I chose to use a GitHub App for authentication
because the alternatives
(like PAT)
didn't feel great from a security viewpoint.
The authentication process is straightforward and not very interesting,
since I just had to follow the [official documentation].
Also, to keep things simple,
the MVP version generates the Installation Access Token (IAT) at each request
instead of caching it.
I plan to take care of this in the next update.

When it comes to sending the `repository_dispatch` event,
the [event documentation] shows that the payload must contain an `event_type`.
Instead of having the server fill it with a predefined value,
a specific `event_type` is required to create a subscription.
This means that a downstream repository can contain
multiple `repository_dispatch` triggered workflows,
but select only some to be triggered for a specific branch update.

<!-- LINKS -->
[official documentation]: https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app
[event documentation]: https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#repository_dispatch

### Hardening the system for resilience

At this point, the main features were all implemented,
but there were still many edge cases to be taken care of.
What if a trigger fails?
What if multiple upstream repositories update at the same time
for the same downstream repository?

The initial MPSC solution to link the polling and the trigger engines
was very fragile.
A simple network error or a server crash
would prevent the delivery of the trigger event
until the repository updates again.
To solve this issue,
I added a new database table to use as a trigger queue.
This way, if the process crashes, the job remains in the queue.
I also implemented a retry mechanism
using an exponential backoff strategy
to mitigate the effect of network errors.

After introducing the trigger queue,
another problem arose:
if two upstream repositories update simultaneously
for the same downstream repository,
how many jobs are queued?
A single workflow should only be triggered once,
but the current implementation queued each branch update as a new job.
I solved this issue by adding an `ON CONFLICT` clause to my database query.
In simple terms, I attempt to queue the job;
however, if there is already an entry
where the target repository and the event type are the same,
the existing job is simply updated.

### Managing technical debt

While developing this project,
I always felt the need to strike a balance
between the advancement of the project
and maintenance to keep a certain level of code quality.
This is important in general,
because if we focus entirely on developing new features,
we end up with a codebase that is difficult to understand and operate on.
That discourages people from continuing to develop it.
On the contrary,
focusing too much on code quality
risks stagnating progress over negligible gains.

Technical debt is the concept that describes this kind of friction.
The idea stems from monetary debt,
but it has some significant differences
since we can't measure it in an absolute way:
instead of having to pay a specific sum of money to a concrete creditor,
we "pay" it by using conceptual and conditional knowledge
about programming and system design
to manage a project
based on an imperfect expectation of the future.

Some types of debt are relatively easy to fix,
like the use of ineffective programming patterns,
but others might originate from much more serious,
non-measurable problems, like a deep architectural flaw.
Technical debt also accumulates interest:
the more you wait to fix it, the more difficult it is to pay it off,
and if it involves public-facing APIs
you're forced to introduce a breaking change to solve it.

Can we pay off all technical debt, or avoid it completely?
No, it would require being able to perfectly predict the future.
However, we can do quite a lot to minimize it.
Since we ultimately pay it with cognitive load,
if we can rely on automation instead of human discipline
to catch bad code,
we are solving a significant part of the problem.
With a CI pipeline that can lint, test and build your code,
and more recently,
AI agents that can semantically review your code before you merge it,
we can often prevent technical debt from ballooning in size.
Another thing we can do to prevent increasing technical debt
is to minimize code coupling:
that way, if a module is problematic,
the debt is strictly contained in the affected module,
and we can easily swap it out with a better one.

Even if those things attempt to _prevent_ technical debt,
some will inevitably slip through,
but that doesn't mean that it should live rent-free in your codebase.
For example, if you're consciously taking a shortcut,
highlight it with a comment
to keep yourself accountable for refactoring it in the future.
Regularly scan your codebase,
and open issues to address architectural problems.
Then, periodically pay off those bills.

I found that using AI in developing a project
is a double-edged sword:
on one hand it is extremely useful for writing tests and refactoring code,
but on the other hand you risk introducing behavior that you don't understand,
especially if you're not sufficiently descriptive.
However, if you understand the limitations
and know how to use it well, it can really accelerate your workflow.

## Future of the project

Version `0.1` is already published, but the project is far from being complete.
I already have a couple more versions on the roadmap.
The next version will be focused on performance and code quality changes.
I will set up a caching system for tokens,
add span-based tracing for observability,
and rate limit outgoing requests to GitHub.
Version `0.3` will focus on feature expansion,
adding support beyond GitHub,
and supporting immediate branch updates
by setting up a webhook in the repository workflow.

I also plan to make a couple of satellite projects written in Go,
since the job market in this language
is at least an order of magnitude greater than the Rust one.
The first one will be a simple CLI to manage subscriptions,
and the second will be a notification service
that can forward workflow results (especially failures) to various platforms,
like Slack or Discord.

## What I would have done differently

Analyzing the git history of this project, I found many high-churn areas.
High-churn is expected in a new codebase;
however, I think it is still important to learn from it
to minimize its occurrence in the next project.

My biggest error didn't affect the diff size much, but was a vision overreach:
I was too ambitious at the beginning, and I started developing a SaaS project.
Then I realized that it wasn't a good idea for a beginner project,
due to the non-trivial issues of deployment, scalability and security,
so I resized it to a single-tenant microservice.
Next time I will try to have a more realistic vision of the project.

When prototyping, I started inlining SQLx queries with the business logic,
but that became a mix of code and queries.
In the end, I had to use AI to make an incremental but extensive refactor
to implement the repository pattern.
Next time I will adopt the repository pattern right away,
instead of fixing it later.

I already knew about the newtype pattern
but I procrastinated on its implementation
because I wanted to focus more on the initial development of the features,
so I heavily relied on basic string types
instead of creating self-validated types on initialization.
This would have allowed cleaner business logic on day-one
and fewer safety bugs.

Ultimately, I learned that prototyping shouldn't be an excuse
for not adopting known patterns and learning about new patterns,
and that it is important to inform yourself
about the complexity of a certain architecture before implementing it.

## Conclusion

I feel quite satisfied with how the project went,
and I'm determined to continue developing it for the learning experience,
even if it is quite a niche use-case.
Anyway, thanks for reading this blog post.
If you are interested in trying it out, you can check out [CommitBridge];
just keep in mind that the project still has some rough edges.

<!-- LINKS -->
[CommitBridge]: https://github.com/Nilirad/commit-bridge
