# A Kafka Anti-Pattern

Kafka is a streaming platform. A streaming platform has three key capabilities:
1. Publish and subscribe to streams of records, similar to a message queue or enterprise messaging system.
1. Store streams of records in a fault-tolerant durable way.
1. Process streams of records as they occur.

At times, in eagerness to standardize, systems with above characteristics (specifically item 3 above) is applied to data migration requirements that have bounded contexts, it results in an anti-pattern. The anti-pattern arises from a simple question:

How does the proposed data migration system using Kafka knows when to stop the automated job (when the last record is processed)?

|  | Design Considerations | Specific Scenarios | Broader<br>design<br>patterns |
|---|----------------------|---------------------|-------------------------------|
| 1 | Job should be complete when the last record is processed | When the last record is processed the job should stop | Bound<br>ed<br>context<br> |
| 2 | Maintain uniqueness and completeness of dataset with respect to when extract was triggered  | Two jobs are run one after the another. The number of records is 100. After each job is finished only 100 records should be processed, not 200 records, the cumulative total | Stateful |
| 3 | Consideration 1 should be satisfied in the event of multiple extracts | Two automated jobs are run one after the another. The number of records at the time of first job is 700 records. The number of records at the time of second job is 702 records. There should be two datasets extracted one with 700 records and 702 records | Temporal<br>ordering<br><br><br>users |
| 4 | Consideration 1 to be satisfied even when multiple extracts are initiated in short span of time and those extract jobs run in parallel (with automation fat-fingers<br>are common occurrences) | Two jobs are started within short span of time and happen to run in parallel. | Idempotency<br><br><br> |
| 5 | Corollary of 2 & 3 is that datasets extracted can be sorted by temporal order | Two different data sets should be extracted based on when the job was triggered irrespective of if the jobs are running in parallel or sequence | Extracts readable by business users |
| 6 | Maintain Idempotency, especially when solution is implemented using end-to-end automation |  |  |
| 7 | Consideration 5 allows for rollback and re-execution. It is a given in Data Migration that things usually do not<br>work in the first attempt |  |  |
| 8 | Allow extracts to be read using excel so that business users can validate results |  |  |


## Comparison of File Based Ingestion and Kafka Stream Based Ingestion


| Solution   | Design elements offered natively | Broader design pattern |
|------------|----------------------------------|------------------------|
| File based | 1. A job can easily identify the last record of a file and stop processing after that record <br>2. Each file extract offers uniqueness and completeness of dataset <br>3. Multiple extracts results in multiple files <br>4. Multiple extracts running in parallel still result in multiple files <br>5. Files can be sorted by timestamp and hence can be sorted by trigger order of job <br>6. Idempotency is maintained by processing the files in the order they were created <br>7. Business users can directly read the extract <br> | * Bounded  :heavy_check_mark: <br>* Stateful  :heavy_check_mark:<br>* Temporal ordering  :heavy_check_mark:<br>* Idempotency  :heavy_check_mark:<br>* Extracts readily readable by business users :heavy_check_mark: |  
| Kafka | *Note:* Kafka operates on un-bounded context on an infinite time window. Further adhoc publishing of full load during each run removes state information from Kafka<br><br>1. Kafka does not offer a way to recognize when a job has started and when a job has finished. Kafka is designed more for shoot and forget requirements. For example, a job can finish at 3:01, 3:02 or 3:03. Kafka can only tell what is the set of records at each of those time intervals, Kafka can not answer if at each of those time intervals, if the data set is complete and unique to a particular job <br><br>2. Kafka does not offer a way to differentiate the boundaries between multiple extracts. Each extract is appended to the previous extract in the log <br><br>3. Multiple extracts running in parallel can result in jumbled data sets. It is not necessary the extracts for the first job triggered are fully appended and then the extract from the second job are appended. The extracts from the multiple job can appear in any random sequence in the log <br><br>4. Json is not directly readable by business users  <br><br>*Caveat:* Kafka is used without any customizations to solve this specific requirement. Any such customization would appear in the table highlighted below | * Un-bounded  :x:<br>* Stateless  :x:<br>* Random ordering  :x: <br>* Idempotency :x:<br>* Extracts not readily readable by business user :x:<br><br>*Note:* Especially when multiple partitions used and when multiple producers(parallel jobs) are in play | 

Using Kafka for Data Migration envisions using Kafka with its design elements(in the given context) of {Unbounded, Stateless, Random Ordering, unknown Idempotency, not readily readable by business users} to a solution that requires design elements of {Bounded, Stateful, Temporal Ordering, readily readable by business users}, thus resulting in an Anti-Pattern. This type of anti-pattern is specifically known as the "Golden Hammer" anti-pattern
