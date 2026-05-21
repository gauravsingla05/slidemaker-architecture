# Case Studies

Real bugs that shipped to users and the diagnostic paths that found
them. Each write-up follows the same shape: symptoms, failing flow,
initial (wrong) theories, root cause, fix, and lessons.

| Case study | Subsystem |
|------------|-----------|
| [The infinite echo loop](echo-loop.md) | Collaboration sync |
| [Late login leaking anonymous identity](late-login-anonymous-id.md) | Collaboration identity |
| [The brand kit color that ate every title](brand-kit-color-override.md) | Theme & brand kit |

## Why write these

A post-mortem is not just for the team that wrote the bug. It is for
the next engineer who hits a similar class of bug, and for readers
trying to understand how the system actually behaves under stress —
which is rarely how the design documents say it behaves.

A good case study leaves the reader with one or two transferable
lessons, not a list of file changes.
