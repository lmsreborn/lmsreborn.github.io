<!--more-->

## Table
- tableEnv: TableEnvironment
- logicalPlan: LogicalNode
- tableSchema
- tableName

### TableEnvironment

### LogicalNode
1. relNode
2. output

### 
## Logic
1. planner.parse(query)
    1. val parser: SqlParser = SqlParser.create(sql, parserConfig)
    2. val sqlNode: SqlNode = parser.parseStmt
    
    - SQLNode
        - operator: e.g Union All
        - operands: 两个语句
        
2. val validated = planner.validate(parsed)
      
3. val relational = planner.rel(validated)// transform to a relational tree
    1. sqlToRelConverter.convertQuery(validatedSqlNode, false, true)



## DataStreamCalc
- translateToPlan
    1. 第一个transformation
    ```
    val inputDataStream =
    getInput.asInstanceOf[DataStreamRel].translateToPlan(tableEnv, queryConfig)
    ```
    2. 第二个transformation
    ```
    inputDataStream.process(processFunc)
    .name(calcOpName(calcProgram, getExpressionString))
    // keep parallelism to ensure order of accumulate and retract messages
    .setParallelism(inputParallelism)
    ```
          
    ### 第一个transformation
    
    ![](/img/trans_1.png)
    ![](/img/trans_1_2.png)
    
    ### 第二个transformation
    
    ![](/img/trans_2.png)
    ![](/img/trans_2_2.png)

    

