# Web Worker Latency Sensitive Hint Explainer

## Authors:

- Wang, Zhibo <zhibo1.wang@intel.com>


## Participate
- github.com/riju/latencySensitiveWebWorker/issues/


## Introduction

[Web Workers](https://html.spec.whatwg.org/multipage/workers.html#workers) brought multithreading to browsers. To keep the UI smooth and responsive, any compute-intensive workload should not be handled on the main (rendering) thread. Till now, we have been using a cookie-cutter approach to offload such “background” jobs to Web Workers. Compute workloads have varied characteristics, some tasks are latency sensitive while some can function better in a consistent level of performance for prolonged periods of time. To satisfy tasks with different performance expectations, modern CPUs incorporate a hybrid architecture with high-performance and high-efficiency cores. Operating systems have APIs to control a thread’s Quality of Service (QoS). Under the hood, hardware schedulering tools look at various performance monitoring units and give hints to the Operating System which makes the decision to deploy the task to a performance core or an efficiency core.

We propose introducing a latency sensitive hint attribute to the Web Workers so that applications and libraries have a way to explicitly label a worker’s preference. User Agents can take this hint to configure platform thread and possibly affect the scheduling policy.

Windows 10 introduced a new QoS level called [EcoQoS](https://devblogs.microsoft.com/performance-diagnostics/introducing-ecoqos/) which is valuable for workloads that do not have a significant performance or latency requirement. Developers can set Power throttling properties on threads using the [SetThreadInformation()](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-setthreadinformation#remarks) API to control a thread’s quality of service (QoS) and enable them to always run in an energy efficient manner. [Apple](https://developer.apple.com/documentation/apple-silicon/tuning-your-code-s-performance-for-apple-silicon#Assign-Quality-of-Service-(QoS)-Classes-to-Work) suggests MacOS developers to use the new setQualityOfService API instead of thread priority and scheduling parameters. Linux introduced an [energy aware scheduler (EAS)](https://github.com/torvalds/linux/blob/master/Documentation/scheduler/sched-energy.rst) for heterogeneous processors to schedule the threads to a most efficient core.

## Goals

The goal is to allow a Web Worker the means to specify a hint if the compute workload they are supposed to perform is latency sensitive or not.

## Non-goals

We do not want to manage the overall scheduling priority or influence any userspace scheduler. [Prioritized Task Scheduling](https://wicg.github.io/scheduling-apis/) is better suited to make sure developers don’t use the frame budget to keep their apps responsive.

We do not want to influence I/O-bound tasks or any other resource loading optimizations browsers do according to their heuristics priority. [Priority Hints](https://wicg.github.io/priority-hints/) is better suited to signal fetch priorities.

## User research
Multiple stakeholders confirmed they would like to offload some of the background processing in the Workers to Efficient Cores to meet their power budgets. Our experiments with using ```ThreadType::kResourceEfficient``` for WebRTC threads influenced the scheduler to use EcoQoS on Windows to opportunistically save power. Initial results showed ~50 mW power savings for Teams Web 1x1 call on Alder Lake based laptop.

## API

When the user agent creates the corresponding native threads for the web workers, also set the quality of service according to the hint given.
The latency sensitive attribute of a worker is only decided by the parameter that construct the instance and is not affected by its parent worker if it has one.
The latency sensitive attribute should not affect the worker’s platform thread priority, to keep away from priority inversion and other scheduling issues.

For worker threads which demand high responsiveness and are sensitive to latency, use ```latencySensitive: 'high'```. This will provide a hint to the underlying system that this Worker can be run on an core which prioritizes Performance.

```js
let worker = new Worker('worker.js', { latencySensitive: 'high'});
```

For worker threads which do not demand high responsiveness and are **NOT** sensitive to latency, use ```latencySensitive: 'low'```. This will provide a hint to the underlying system that this Worker can be run on an core which prioritizes Efficiency.

```js
let worker = new Worker('worker.js', { latencySensitive: 'low'});
```
Sometimes we are not confident about the exact latency sensitivity of the workload. Our guidance is to use ```latencySensitive: 'default'```. This will provide a hint to the underlying system that this Worker does not actually have a preference. User Agents can use their policy like EfficiencyMode (used in Edge) and other factors on power and thermal throttling (system on power saving mode, plugged in or in battery, etc) to determine the default behaviour.


```js
let worker = new Worker('worker.js', { latencySensitive: 'default'});
```

If there is no global policy on the system, UAs should try to treat them as **NOT** latency sensitive and hint the underlying systems to prioritize efficiency over performance.


## Key scenarios


Tasks on web workers are usually time consuming and not latency sensitive. Tagging these workers as "low" latency sensitivity will avoid competing for computing resources with other threads and will possibly improve the power efficiency of running those tasks. Such as:
- Data prefetching and preprocessing
- Caching and local storage
- Compressing and data integrity validation
- Spell check

Some work may provide the user with a real-time result, where a "high" latency sensitivity tag would indicate the platform to provide the best performance and the shortest latency despite platform default configuration. Such as:
- Audio processing in real-time communication



## Detailed design discussion

- Do we need to change the ```latencySensivity``` attribute while the task is ongoing ?
- Do we need getter / setter or an onAttributeChange() event to signal the changes ?
- Would the cost of switching tasks midway to different cores affect performance.

```js
// Illustrated with example code.
```


## Considered alternatives

### [Prioritized Task Scheduling](https://wicg.github.io/scheduling-apis/)

[Prioritized Task Scheduling](https://wicg.github.io/scheduling-apis/) is a good way to yeild to the browser and make sure the app is responsive by not crossing the frame budget. We want to deal only with the WebWorkers that work on the background and not with user-blocking or user-visible tasks. Also we do not want to define any priorities for tasks, just hints to utilize the CPU more efficiently.


### [Priority Hints](https://wicg.github.io/priority-hints/)

[Priority Hints](https://wicg.github.io/priority-hints/) is better suited to signal fetch priorities and other I/O bound work done in the background. We want to focus on only the "compute" bound work done by WebWorkers.

## Stakeholder Feedback / Opposition


- [Google] : Positive
- [Zoom] : Positive.


## References & acknowledgements

Many thanks for valuable feedback and advice from:

- Rijubrata Bhaumik
- Jianlin Qiu
