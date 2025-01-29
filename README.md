# goit-de-hw-04
The repository for the 4th GoItNeo Data Engineering homework

The difference in the number of jobs we observed in Spark UI for each script is due to how Spark optimizes the execution plan and how actions are applied in a "lazy execution" manner. Let see in more detail what do we have for each case:

1. First Script (5 Jobs)

```
nuek_df = spark.read.option("header", "true").option("inferSchema", "true").csv('./nuek-vuh3.csv')
nuek_repart = nuek_df.repartition(2)
nuek_processed = nuek_repart.where("final_priority < 3").select("unit_id", "final_priority").groupBy("unit_id").count()
nuek_processed = nuek_processed.where("count>2")
nuek_processed.collect()
```
**Explanation:**

In this case, we are performing transformations and only one _collect()_ action at the end.
When we perform this action, Spark evaluates the transformations that were lazily constructed before it. It creates a DAG (Directed Acyclic Graph) for the full set of transformations to execute.

**In total:**
_Job 1_: Loading the data from the CSV.
_Job 2_: Repartitioning the data.
_Job 3_: Filtering by final_priority < 3.
_Job 4_: Grouping the data by unit_id and applying count.
_Job 5_: Filtering the results by count > 2 and collecting.

**Result:** A total of 5 jobs. Since we are using lazy execution and only triggering one action (collect()), Spark only executes one DAG (with a set of transformations inside it), running 3 jobs in total to apply transformations, and 2 jobs for reading and repartitioning.

2. Second Script (8 Jobs)

```
nuek_processed = nuek_repart.where("final_priority < 3").select("unit_id", "final_priority").groupBy("unit_id").count()
nuek_processed.collect() # First collect action
nuek_processed = nuek_processed.where("count>2")
nuek_processed.collect() # Second collect action
```
**Explanation:**

In this script, we have two _collect()_ actions. For each _collect()_, Spark needs to run all transformations from scratch and compute the results again.

The first collect runs the DAG for reading data, repartitioning, transforming, and then collecting the result.
After the first _collect()_, we perform an additional transformation (the filtering where("count>2")).
Since the second _collect()_ is a new action:
Spark needs to redo all the previous transformations (creating a new DAG), and it triggers another set of jobs.

**Result:** A total of 8 jobs. The first set of jobs runs the transformations and collects the result, and the second set redoes those steps and collects the result again.

3. Third Script (7 Jobs with Caching)

```
nuek_processed_cached = nuek_repart.where("final_priority < 3").select("unit_id", "final_priority").groupBy("unit_id").count().cache() 
nuek_processed_cached.collect()  # First collect action
nuek_processed = nuek_processed_cached.where("count>2")
nuek_processed.collect()  # Second collect action
nuek_processed_cached.unpersist()  # Removing cached data
```

**Explanation:**

The key difference here is that we cache the results of the first transformation. This means that after the first _collect()_, the intermediate results are stored in memory.

When we perform the second _collect()_, Spark doesn't need to re-run the expensive transformations because they were already computed and cached in memory. So instead of running the complete DAG again, Spark simply retrieves the results from memory and applies the second where("count>2") filtering operation.
This is why the second _collect()_ results in fewer jobs:
a) The transformation is not recomputed because the result is cached.
b) Only one additional job is needed for filtering.

**Result:** A total of 7 jobs. The caching mechanism allows Spark to optimize and avoid recomputing the transformations after the first _collect()_.

**Summary of Job Counts:**
**5** Jobs in the first script because only one _collect()_ action was used.
**8** Jobs in the second script because two _collect()_ actions required Spark to recompute all transformations from scratch for each action.
**7** Jobs in the third script because caching reduces the need to recompute transformations, making the second collect faster and causing only one additional job for the second filtering.

In short, using actions like c_ollect()_ triggers Spark to compute the transformations and materialize results, while caching helps improve performance by preventing redundant calculations across multiple actions.