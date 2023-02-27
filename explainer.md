# Web Worker Quality of Service Explainer

## Authors:
- [Wang, Zhibo](https://github.com/zhibo1-wang)


## Participate
- https://github.com/riju/web-worker-quality-of-service/issues/


## Introduction
[Web Workers](https://html.spec.whatwg.org/multipage/workers.html#workers) enable multithreading in web browsers. By offloading compute-intensive workload to worker threads, we can achieve better UI smoothness and responsiveness. Till now, we have been using a cookie-cutter approach to offload such "background" jobs to Web Workers. Compute workloads have varied characteristics, some tasks are latency sensitive while some can function better in a consistent level of performance for prolonged periods of time. To satisfy tasks with different performance expectations, modern CPUs incorporate a hybrid architecture with high-performance and high-efficiency cores. Operating systems have APIs to control a thread's Quality of Service (QoS). Under the hood, hardware schedulering tools look at various performance monitoring units and give hints to the Operating System which makes the decision to deploy the task to a performance core or an efficiency core.

We propose introducing a quality of service attribute to the Web Workers so that applications and libraries have a way to explicitly label a worker’s preference of performance. User Agents can take this hint to configure platform thread and possibly affect the scheduling policy.

Windows 11 introduced [EcoQoS](https://devblogs.microsoft.com/performance-diagnostics/introducing-ecoqos/) which is valuable for workloads that do not have a significant performance or responsiveness latency requirement. Developers can set Power throttling properties on threads using the [SetThreadInformation()](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-setthreadinformation#remarks) API to control a thread's quality of service (QoS) and enable them to always run in an energy efficient manner. Apple [suggests](https://developer.apple.com/documentation/apple-silicon/tuning-your-code-s-performance-for-apple-silicon#Assign-Quality-of-Service-(QoS)-Classes-to-Work) MacOS developers to use the new `setQualityOfService` API instead of thread priority and scheduling parameters. Linux introduced an [energy aware scheduler (EAS)](https://github.com/torvalds/linux/blob/master/Documentation/scheduler/sched-energy.rst) for heterogeneous processors to schedule the threads to a most efficient core.

## Goals
The goal is to allow a Web Worker the means to specify a quality of service hint, which specifies how the compute workload should be run.

## Non-goals
We do not want to manage the overall scheduling priority or influence any user-space scheduler. [Prioritized Task Scheduling](https://wicg.github.io/scheduling-apis/) is better suited for scheduling asynchronous tasks within the thread, and to make sure developers don't use the frame budget to keep their apps responsive.

We do not want to influence I/O-bound tasks or any other resource loading optimizations browsers do according to their heuristics priority. [Priority Hints](https://wicg.github.io/priority-hints/) is better suited to signal fetch priorities.

## User research
Multiple stakeholders confirmed they would like an option to offload some of the background processing web worker workload to efficient cores to meet their power budgets. 
Experiments shows that utilizing efficient scheduling would have significant power saving on heavy-loaded background workers, including AI pre-processing in web real-time communication.

## API

```idl
partial dictionary WorkerOptions {
  WorkerQualityOfService qualityOfService = "default";
};

enum WorkerQualityOfService { "high", "low", "default" };
```

When the user agent creates the corresponding native threads for the web workers, also set the quality of service or call related platform interface according to the hint given.
The quality of service attribute of a worker will be decided by the parameter that construct the worker instance and is not affected by its parent worker if it has one.
The quality of service attribute should not affect the worker's platform thread priority, to avoid priority inversion and other scheduling issues.

For worker threads which demand high performance, responsiveness, or sensitive to latency, use `qualityOfService: 'high'`. This will hint the platform that the worker thread can be scheduled or optimized for best performance.
```js
let worker = new Worker('worker.js', { qualityOfService: 'high' });
```

For worker threads which do not demand performance, responsiveness and are **NOT** sensitive to latency, or expect its runtime energy efficiency, use `qualityOfService: 'low'`. This will hint the platform that the worker thread can be scheduled or optimized for best efficiency.
```js
let worker = new Worker('worker.js', { qualityOfService: 'low' });
```

If prioritizing performance or efficiency is not necessary for the worker thread, it is recommended to use the option `qualityOfService: 'default'`. This option instruct the platform to use default settings for the worker thread. The final behavior of the thread will be determined by performance-efficiency policies of the user agent, like efficiency mode in the Edge, as well as platform power and thermal throttling policies, which will take into account factors such as system power saving mode, power status, and battery percentage.
```js
let worker = new Worker('worker.js', { qualityOfService: 'default' });
```

If `qualityOfService` is `undefined` or other values, it would be interpreted as `'default'`, in which case user agents should use a system default setting.

## Key scenarios
Tasks on web workers are usually time consuming and not sensitive to responsiveness. Tagging these workers as "low" quality of service will prevent them from competing for computing resources with other threads and will possibly improve the power efficiency for running those tasks. Examples are:
- Data prefetching and preprocessing
- Caching and local storage management
- Compressing and data integrity validation
- Spell check

Some work may be responsible of providing the user with a real-time result, where a "high" quality of service hint would instruct the platform to provide the best performance and responsiveness. Such as:
- Audio processing in real-time communication

## Detailed design discussion
- Do we need to change the `qualityOfService` attribute while the task is ongoing?
  - Will there be a performance issue?
- Do we need getter, setter, and an onchange event?

## Considered alternatives

### [Prioritized Task Scheduling](https://wicg.github.io/scheduling-apis/) for JS main thread
[Prioritized Task Scheduling](https://wicg.github.io/scheduling-apis/) is a good way to yield to the browser and ensure the application's responsiveness. Our focus is to deal with the Web Workers that work on the background as a separate thread.

### HTML [Priority Hints](https://wicg.github.io/priority-hints/)
[Priority Hints](https://wicg.github.io/priority-hints/) is a HTML attribute to signal the user agent with priorities for fetching and I/O bound works. We want to focus on the compute bound work done by Web Workers.

### Platform thread priority hints
We are intended to avoid introducing the platform thread priority hint as an alternative or along with the quality of service hint, due to following reasons:
- In repesct of our performance/efficiency target, the platform thread priority is not necessary.
- Letting javascript control the platform thread priority may have permission risks.
- If we do so, whether we should or not, and how to define the priority for nesting workers would be a problem.

## Stakeholder Feedback / Opposition

Bikeshedding : 
```qualityOfService``` vs ```computePriority``` 

## References & acknowledgements
Many thanks for valuable feedback and advice from:

- [Rijubrata Bhaumik](https://github.com/riju)
- [Jianlin Qiu](https://github.com/taste1981)
- [François Beaufort](https://github.com/beaufortfrancois)
- [Youenn Fablet](https://github.com/youennf)
