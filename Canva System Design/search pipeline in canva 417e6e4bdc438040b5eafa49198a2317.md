# search pipeline in canva

![image.png](search%20pipeline%20in%20canva/image.png)

complete pipeline

![image.png](search%20pipeline%20in%20canva/image%201.png)

Model of a search pipeline as a series of ordered and isolated steps, grouped within phases

![image.png](search%20pipeline%20in%20canva/image%202.png)

Steps in the search pipeline and which phases they belong to

1. **Isolated**: Each component can read the output from previous steps, but it can't see other components participating in the current step. Advantage? Parallelism.
2. **Stateless**: Each component should manage state out-of-process (for example, in Redis or DynamoDB).
3. **Transient**: Components are not persistent or durable, which re-enforces the stateless property described previously.

The execution of the search pipeline is:

- **Ordered**: Each step occurs sequentially, components in each step either execute sequentially or in parallel as described in the table above.
- **Immutable**: It's not possible for steps to mutate state from previous steps.

<u>**Validation:**</u>  The pipeline can make use of validation components. Should any fail, the pipeline is halted. Validations are expected to be fast and ordered such that the fastest failing validators run first. Example: page numbers for reasonable constraints on the starting index, page size.

<u>**Rewriting:**</u> Rewriting is a highly-opinionated and lossy step. Types include:

1. A *transparent* rewrite preserves the original query, either by performing the rewrite to a different field, or overwriting the original query field, and copying the original query to another field. Downstream steps are expected to use the rewrite but have the option of referring to the original query, although this practice is somewhat discouraged
2. An *opaque* rewrite doesn't preserve the original query, thus forcing all downstream steps to use it.
3. A *forked* rewrite splits into multiple branches, one for each rewrite output. 

<u>**Tokenization:**</u> Given a rewritten query, the goal of **tokenization** is to deconstruct the query text into smaller atomic text units that can be individually processed. Tokenization techniques: locale, semantic, search syntax.

<u>**Annotation:**</u> An annotation is metadata pertaining to the search query. We use annotations to provide additional query intent or understanding derived from the existing query context, often from external systems. Examples: Name-Entity Recognition, Language Detection, Sensitive/Problematic classified queries, Synonyms. The search pipeline supports annotations at two levels:

- **Token Level**: Annotations on a per-token basis.
- **Query Level:** Annotations that apply to the whole search query (with no reference to tokens) or annotations that incorporate more than one token (multi-token annotations).

<u>**Candidate Generation:</u>**  A ****candidate generator interprets an annotated query and formulates a request to a search engine or data store to retrieve results. Each candidate generator can interpret the annotated query differently, allowing us to make tradeoffs across recall and retrieval options. We can register multiple candidate generators and have the pipeline coordinate the execution. This allows us to introduce various timings or result-count deadlines. The results from each candidate generator and the resulting metadata is merged into an in-memory data structure for re-ranking. We use various data stores, including: OpenSearch, Solr, and SageMaker and now exploring vector data stores.

<u>**Feature extraction and enrichment:**</u> We're in the process of building a high-performance feature store backed by Redis. In the meantime, we call into DynamoDB for these additional features. A feature extractor is a component that, provided a given (query, document) pair, outputs one or more features. Some feature extractors might only depend on the query, document, or both. 

<u>**Re-ranking:**</u> The re-ranking step allows for fine-tuning the final order of candidates before returning them to the caller. This phase allows us to incorporate business logic and machine learning models better suited to post-fetch evaluation. This approach allows us to use [**BM25**](https://en.wikipedia.org/wiki/Okapi_BM25) search, while re-ranking through dense vector spaces. Multiple different re-ranking components: business-rules, recommendation and personalization, utilities.

<u>**Overfetching and Bots:**</u>  We take different approaches to conducting a search depending on the requested page number and size. For the first set of 500 results, we prioritize precision, by running the full query understanding and re-ranking components, making heavy use of caching to ensure performance. We then slice into that result set for the first pages, allowing us to present the user with the best possible items on the first page ranked from the top 500 results. For results beyond the scope of this overfetched set, we adopt a faster and more efficient pipeline, prioritizing speed over precision. The rationale is that very few users interact with results past around position 200, and the likelihood of us serving results to a bot is quite high.

<u>**Fault tolerance:**</u>  

1. **Limiting Blast Radius**: Should a component encounter or generate an exception, it's responsible for limiting its own blast impact on the pipeline. For example, if an annotator generates annotations that could be considered optional for candidate generation, the annotator should handle exceptions gracefully, return nothing, and log as appropriate.
2. **Deadlines**: The pipeline could implement deadlines on various phases, steps, or components. Strategies include producing as many results as possible within a specified timeframe.
3. **Retries**: Should a component fail or throw an exception, the pipeline could automatically retry that component should the error condition indicate a transient issue.
4. **Circuit breakers**: We can create chains of components that execute according to failover rules.