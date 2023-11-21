# Spark

## Spark optimization
1. RBO: Rule based optimization
2. CBO: Cost based optimization
   * Pros: cost based.
   * Cons: maybe the worest plan
3. AQE: Adaptive query execution
   * The optimization happens during the whole execution based on the statistics of each stage.
  
     
## Why it is faster than MapReduce? 
1. Memory VS. Disk
2. DAG supported
3. Auto optimization
