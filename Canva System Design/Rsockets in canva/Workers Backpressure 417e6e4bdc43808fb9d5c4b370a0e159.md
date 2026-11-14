# Workers Backpressure

*How we taught Canva's queue workers to slow down when dependencies fail, then speed back up on their own.*

We needed an adaptive worker backoff solution general enough for our fleet of diverse queues and named it **Worker Backpressure**: a mechanism built into our `queue` library. The worker watches how its own work is going and adjusts its speed accordingly, easing off as errors climb and ramping back up as the dependency recovers, with no human intervention required.

Backpressure is a **feedback loop** around the worker's calls to its dependency. It tracks the outcome of each call as a signal of the dependency's health, and regulates the worker's concurrency: how many messages it may process at once. When the dependency looks healthy, backpressure stays out of the way; when it struggles, it throttles the worker.

Backpressure consists of three pieces:

1. **Signals.** After each message is processed, the worker records the outcome: success or failure.
2. **Backpressure Controller.** A pluggable controller (an interface rather than one fixed algorithm) consumes those outcomes and maintains a single number: the **backoff factor**, ranging from `0.0` (full speed) to `1.0` (fully backed off). It works against a configured **set point**: the failure rate the controller treats as acceptable background noise. While the failure rate stays below the set point, the controller does nothing. Once the rate climbs above it, the controller starts backing the worker off.
3. **Permits.** Before each poll, the worker asks the controller how many messages it may pull and process concurrently. The controller scales that number down in proportion to the current backoff.

![image.png](../Worker%20backpressure/image.png)

Worker extended with Backpressure mechanism

An important design choice is that all of this happens locally, with no external coordinator and no added network calls. The entire runtime cost is two arithmetic operations: one to move the backoff factor after each outcome and one to scale the requested concurrency at each poll.