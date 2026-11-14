# Canva System Design

[Resources](Canva%20System%20Design/Resources%20414e6e4bdc438074a682de7da6022998.md)

1. **Autocorrection  System:**
    1. started off with how to distinguish between a misspelled word and a correctly spelled word. A metric called the Edit/Levenshtein Distance FN_EDIT(wordA, wordB) returns a number that specifies the minimum number of changes required to transform one string to the other.
    2. The first step was to identify and extract query chains across users, so for example if we had a user that made a series of queries (archery, archer, aarow, arrow) in a session with Canva, we would store those queries. With these queries we were able to strip out chronologically ascending pairs such as (archery, archer), (archery, aarow), (archery, arrow) followed by (archer, aarow) and so on.
    3. To achieve a high signal that the latter query in each pair is a valid correction, we also added a conversion metric to each of these queries (i.e. whether the query converted to clicks or revenue). With the conversion metric added to the query we were now able to add a probability score to each query.
        
        > **P(conversion|query) = number of times the query resulted in a conversion / total number of times the query was made.**
        > 
        
    4. Data extraction to fetch correction entries:
    
    ![image.png](Canva%20System%20Design/image.png)
    
     e.  To add more signals to the value of our pairs, the frequency of times a pair occurred was also considered to be an important metric. Pairs that occurred only once within the window of a month were dropped out from the final list.
    
     f.  We were able to run the data extraction as described above with an extraction window size of 2 months resulting in close to 32000 corrections for one of our locales
    
    g.  we plan to use the rejection data captured from user events to demote corrections to suggestions instead and if they still perform poorly, remove them from the configuration altogether.
    
    h.  Our data extraction currently runs in a jupyter notebook and most of the logic is single threaded thus limiting the size of our window. We are currently planning to run the data extraction job with Apache Spark so that we can extract and compute most of the logic in parallel which will also enable a bigger window of extraction for user generated events (dated 2019)
    
2. <u>**Elastisearch (since 2022)**</u>: 
    1. **security:** security group level rules, cluster policies for IAM, **fine-grained access control** on cluster, requests must be signed using **Sigv4.**
    2. **leaders and state management:** handled by master nodes, can be run on same or a dedicated instance.
    3. **defining shard and placing nodes:** shards defined on index creation or by index template, shards are placed on nodes based on a replication value/index creation/index template, shard placement avoids adding replicas of the same primary shard on one node instance.
    4. **index mapping and templates:** create indexes using index template but changing the template does not change the configuration of a live index unless settings and mapping of a template are applied directly to that index. 
    5. **indexing:** dynamic fields by default or can be set, supports wildcard field types, required and multivalued fields are not enforceable, routing to shards is calculated based on `_id` field.
    6. **queries:** API with structured JSON request, uses DSL on Lucene, can create query templates that have default values.
    7. **nested data type mapping:** Objects mapped this way capture structured data, and can be queried both individually or as a collection of fields.
    8. **vectors:** Our migration to Elasticsearch unlocks some vector search options with the ability to map fields as dense and sparse vectors, by encoding vectors as binary doc_values.
    9. Elasticsearch doesn't have a way to specify that all documents require a given field, or that a given field must have only one value, for that you would need an ingest pipeline with a fail processor.
    10. You cannot comment JSON and must find a new place for annotations.
    11. If an Elasticsearch cluster is yellow, for example, you must debug this without the graph of nodes and their respective shards with their current status.
    12. You can upload extra files, such as those that contain synonyms, to a collection and reference them by `id`. This `id`, however, is set on upload and isn't mutable, or specifiable.

1. **Endpoint vulnerability management at scale**

Data flow visualisation:

![image.png](Canva%20System%20Design/image%201.png)

To obtain application data, we rely on SentinelOne, our Endpoint Detection and Response (EDR) solution. The EDR agent has a module that takes inventory of all applications installed on all endpoints in the fleet. It aggregates that data, making it viewable on the console and available to query through APIs. Among the data points gathered by the EDR, we get the application name, device identifier, application version, and application vendor. When we upload this data to the console, our EDR vendor assigns vulnerabilities to each application-version combination. We then get the following additional data points through the console or the API: assigned Common Vulnerabilities and Exposures (CVE) ID, Common Vulnerability Scoring System (CVSS) score, date detected, and date last seen.

---

[Microservices in canva](Canva%20System%20Design/Microservices%20in%20canva%20417e6e4bdc4380b98bcdf2a2c9d27962.md)

[Rsockets in canva](Canva%20System%20Design/Rsockets%20in%20canva%20417e6e4bdc4380efb108e7fda356dfd8.md)

[search pipeline in canva](Canva%20System%20Design/search%20pipeline%20in%20canva%20417e6e4bdc438040b5eafa49198a2317.md)

[RDBMS with AWS DMS](Canva%20System%20Design/RDBMS%20with%20AWS%20DMS%20417e6e4bdc4380a6ad55c56477c17873.md)

[Scalability](Canva%20System%20Design/Scalability%20417e6e4bdc4380e1845ac52b0e42fdd0.md)

[Realtime Mouse Pointers](Canva%20System%20Design/Realtime%20Mouse%20Pointers%20417e6e4bdc438031aa0ec89a20fc3e6c.md)

[Session revocations at scale](Canva%20System%20Design/Session%20revocations%20at%20scale%20417e6e4bdc438082907be2d8743b6d20.md)

[Worker backpressure](Canva%20System%20Design/Worker%20backpressure%20417e6e4bdc43806bb8c5c82129d02fba.md)

[CI builds at Canva](Canva%20System%20Design/CI%20builds%20at%20Canva%20417e6e4bdc438060a9add6929c8a828c.md)

[UI/UX optimizations](Canva%20System%20Design/UI%20UX%20optimizations%20417e6e4bdc4380ce973afad036a53979.md)