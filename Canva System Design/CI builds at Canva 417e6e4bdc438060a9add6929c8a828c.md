# CI builds at Canva

It is a decently sized distributed system, non-trivial workload, too many downstream dependencies, and CI performance bound by the longest stretch of dependent actions.

![image.png](CI%20builds%20at%20Canva/image.png)

Canva CI workflow

The working group found and fixed many easily solved problems, such as:

- Preventable Bazel cache issues with Docker images and build stamping.
- De-duplication of Bazel targets across pipelines.
- Fixing pipeline structures by merging or removing steps where applicable.

We also started some longer-term investigations and initiatives, including:

- A pipeline audit to better understand its steps and identify the big improvement opportunities.
- The Developer Environment and Test Platform teams began converting backend integration tests into [**Hermetic**](https://bazel.build/basics/hermeticity) Bazel tests to take advantage of Bazel’s caching and parallelism.
- The Build Platform team started exploring Bazel Remote Build Execution ([**RBE**](https://bazel.build/remote/rbe)) to distribute our Bazel workloads more efficiently.
- The Source Control team was improving the performance of Git repository checkouts and caches.

Bazel promises fast, incremental, and correct builds by providing features such as sandboxing, parallelism, caching, and remote execution. However, there were many challenges in fulfilling these promises. Bazel:

- Is slow to start.
- Adds significant execution overhead because of process sandboxing
- Is hard to migrate builds and tests into because you must declare every input that a rule depends on
- Is hard to make “Remote Build Execution” (RBE) compatible because it requires the worker pool to provide the same inputs to the actions as local execution would

At Canva, we have a framework called the Product Development Process (PDP), which gives us great tools to shape, build, refine, and launch products.

![image.png](CI%20builds%20at%20Canva/image%201.png)

At Canva, we’ve long had a Bazel cache shared across all CI workers. It’s a simple setup but amazingly effective. It’s a service called **bazel-remote**, installed on every instance, backed by an S3 bucket as its storage. So, the gRPC communication between the Bazel server and the cache happens locally, but it’s supported by global shared cache storage. While experimenting with large instances, we learned 2 things:

- We downloaded hundreds of GBs worth of artifacts (mostly containers) on every backend build.
- Bazel has a mechanism called *build without the bytes* (BwoB). It has a `-remote-download-minimal` flag that prevents Bazel from downloading the result of an action from the cache unless another action (like a test) needs it. So we knew it had the potential to save a lot of network I/O and time.

We made a proof-of-concept (PoC) in Bazel to see if it could be simpler and faster. In this new generator, we declare the pipeline configuration in **Starlark** (Bazel’s configuration language), which we convert to YAML, as Buildkite expects.

It resulted in a significantly simpler pipeline generator than its Typescript counterpart because the configuration significantly limits the complexity (that is, there are no side effects, and it’s **hermetic by design**) we introduce into the pipeline definition. The PoC ended up being the base for our current pipeline generator.

The Developer Runtime team developed a framework for hermetic container orchestration using the `TestContainers` library, allowing each test to control its distinct set of storage requirements within the confines of a Bazel sandbox, ultimately allowing us to cache these tests.

<u>**Service container tests and hermetic E2E:</u>**  The Verify Platform team extended this hermetic container orchestration framework to cover Canva backend service containers, with each service having its own TestContainer implementation and associated Service Container Test (a test validating that the container can launch correctly). This shifted many common deployment failures to CI, and is a significant step towards deprecating the non-hermetic full-stack tests.

We then used these service container definitions to compose hermetic E2E test environments, allowing E2E tests to be cached, with rebuilds required only when modifying a service involved in the test. The reliability of the new E2E framework also significantly improved over the previous iteration because each service container is hermetically tested in isolation, making failures much easier to detect and diagnose.

Removing `bazel-diff` and `bazel query` from the pipeline generation process reduced the pipeline generation step to 2-3 minutes. However, 2 minutes on the critical path isn’t great, and we knew we could do better.

Brainstorming the problem, we decided we could:

- Generate the pipeline statically instead of at job runtime.
- Avoid the git checkout and preparation time and cost.
- Move the git conditional evaluations to the job runtime.
- See how we could reintroduce `bazel-diff` at runtime.

We quickly had solutions for points 1, 2, and 3, and on 9 January 2024, we launched them for `BE/ML Pipeline`, and soon after, for `FE Pipeline`.

We found that about 80% of the time, our E2E tests were on the critical path of our BE/ML Pipeline, while integration tests were on the critical path for our FE Pipeline.

We improved the E2E tests by fixing their CPU requirements and adding a more effective worker pool to execute them. This reduced E2E test runtimes by 7-10 minutes. We launched on 1 February 2024.

# **Bazel Remote Build Execution (RBE)**

Bazel can distribute its workload across a large cluster of executors. That way:

- Build actions from CI (and developer machines) are sent to a centralized build service, allowing us to distribute build computation across a farm of machines optimized to their workloads.
- Build inputs and outputs are co-located with build workers, allowing for faster I/O.
- We get better sandboxing and isolation (docker containers), allowing for a higher cache hit rate.
- Agent warm-ups were quickly solved using EBS snapshots to load caches into new agents, resulting in significantly shorter wait times.