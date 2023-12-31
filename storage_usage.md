1. **Background**

   **Where does the `show accounts` cmd come from?**
   1. internal

      the CN MetricStorageUsage cron task gathers any new accounts every minute and all accounts every 15 minutes, and then writes them in metrics.
      but this cron task only exists in exactly one CN. 
      
   3. users

      users can launch `show accounts` from the SQL client at any time.

    <br>

   **Who has the access right?**
   1. system account:

      at every minute, the sys account executes `show accounts like xxx` to collect storage usage of newly added accounts.

      at every 15 minutes, the sys account executes `show accounts` to collect storage usage of all accounts
      
   3. normal account:

      any normal accounts can launch `show accounts` SQL, the effect equals to the sys account executes `show accounts like xxx`.

    <br>

   **Storage Usage Definition**

   we only consider these data that are ready to flush to remote or already persistent in remote.

<br>
<br>

![image](https://github.com/gouhongshen/docs/assets/26336316/024759e9-e1e9-4106-8913-100b70af2b2a)

<br>

2. **The Current Implementation**
   
    using `show accounts` to get all storage usage info for all accounts.

    ```
    mysql> show accounts;
    +--------------+------------+---------------------+--------+----------------+----------+-------------+-----------+----------+----------------+
    | account_name | admin_name | created             | status | suspended_time | db_count | table_count | row_count | size     | comment        |
    +--------------+------------+---------------------+--------+----------------+----------+-------------+-----------+----------+----------------+
    | tenant_test  | admin_1    | 2023-10-06 02:43:34 | open   | NULL           |        5 |          45 |      1157 |    0.066 |                |
    | tenant_2     | admin_2    | 2023-10-06 02:44:21 | open   | NULL           |        5 |          45 |      1157 |    0.066 |                |
    | tenant_3     | admin_3    | 2023-10-06 02:44:33 | open   | NULL           |        5 |          45 |      1157 |    0.066 |                |
    | sys          | root       | 2023-09-28 09:44:41 | open   | NULL           |        7 |          82 |  32351824 | 3555.283 | system account |
    +--------------+------------+---------------------+--------+----------------+----------+-------------+-----------+----------+----------------+
    4 rows in set (0.03 sec)
    ```
    <br>

    **How does ****`show accounts`**** work?**

    after the frontend receives `show accounts`, it starts a transaction.

    the first step is to get related account info:
   
   ```SQL
   SELECT
   	account_id AS `account_id`,
    	account_name AS `account_name`,
    	created_time AS `created`,
    	status AS `status`,
    	suspended_time AS `suspended_time`,
    	comments AS `comment`
   FROM
    	mo_catalog.mo_account %s;
   ``` 

    for every account, it uses the built-in functions `mo_table_rows` and `mo_table_size` to get account storage usage info, and then calculates the results with in-memory writes to get the final result.

    ```SQL
    SELECT
        mu2.user_name AS `admin_name`,
        COUNT(DISTINCT mt.reldatabase) AS `db_count`,
        COUNT(DISTINCT mt.relname) AS `table_count`,
        SUM(mo_table_rows(mt.reldatabase, mt.relname)) AS `row_count`,
        CAST(SUM(mo_table_size(mt.reldatabase, mt.relname)) / 1048576 AS DECIMAL(29, 3)) AS `size`
    FROM
        mo_catalog.mo_tables AS mt
    JOIN
        (
            SELECT
                MIN(user_id) AS `min_user_id`
            FROM
                mo_catalog.mo_user
        ) AS mu1 ON mu2.user_id = mu1.min_user_id
    JOIN
        mo_catalog.mo_user AS mu2 ON 
    WHERE
        mt.relkind != '%s' AND mt.account_id = %d;
    ```
    <br>

    **How much does the `show accounts` cost?**

    the cost depends on the number of accounts and how many tables an account has:
      1. the `join` operation could be time-consuming
      2. massive subscriptions could degrade CN performance
   ```vim
   mo_table_size
   mo_table_rows
    |
    |
    |-- tb_1  --
    |-- tb_2     \
    |-- tb_3       ====> UpdateBlockInfos ===> SubscribeTable
    |            /
    |-- tb_n  --
   ```

<br/>

3. **The Optimization Plan**

    in summary, using the checkpoint process, we can save CN's performance.
    <br>
    <br>

    **Checkpoint Overview**
    ```vim
    catalog
    |
    ---------------------------------------------------------
            |               |               |       ...
         db_entry        db_entry        db_entry
                           |
                           |
                          ----------------------------------------
                                    |        |         |     ...
                                tb_entry  tb_entry  tb_entr
                                    |
                                    |
                                   --------------------------------
                                            |         |        ...
                                        seg_entry  seg_entry
                                            |
                                            |
                     ------------------------------------------------------------------------------
                            |       |         |       |        |      |       |         |      |
                           blk     blk       blk     blk      blk    blk     blk       blk    blk
                       |---------------|    |-----------|    |--------------------|
                       s1    ckp1     e2    s2   ckp2   e2   s3      ckp3        e3
               |----------------------------------------|
               -∞             global ckp                e2
    ```

Whenever a block is created or deleted, its metadata is recorded in the catalog. 
The checkpoint runner periodically collects this metadata and logs it in the checkpoint.

<br>

**Implementation Detail**

so we also could record the table size and row counts changing increasingly, 
we only need to add an extra batch to ckp, like:
   
 ```Go
    type AccountInfoBatch struct {
        // { accout_id, database_id, table_id, table_total_flushed_size }
	    Attrs   []string 
	    Vecs    []Vector
	    //...
    }
```    

when the user wants to know its storage usage, it requests CN --> TN, 
and TN returns a global ckp and a list of incremental checkpoints. 
the CN needs to decode these batches stored in checkpoints and combine them with the blocks that are ready to flush.
the codes like:
```python
for ckp in [global_ckp, incremental_ckps] {
    for bat in ckp {
        size[bat.acc_id] += bat.size
    }
}
    
for blk in blk_not_checkpoint_yet {
    size[data.acc_id] += data.size
}    
    
```
<br>

**Interface Compaitable**:

result interface
```Go
// res batch
type Batch struct {
    // account_name     
    // admin_name	
    // created          
    // status           
    // suspendedTime    
    // comment
    // db_count	
    // table_account	// join mo_user, mo_tables, mo_account

    // size (MB)	// comes from ckp
    Attrs []string

    Vecs  []*vector.Vector
}
```
<br>

**Handle Pipeline**
* we can use the log tail pull pipeline
  ```
  OpCode_OpGetLogTail
  ```
* we can use the debug pipeline (dedicated for mo_ctl)
  ```
  TxnMethod_DEBUG
  ```

 
 ```
 type SyncLogTailResp struct {
    CkpLocation []Location
    Commands    []BlockEntry
 }
 ```

  <br/>

 **Others Considerations**
    1. timeliness

       by default, will schedule increment ckp each 5s and global ckp each 100 increment ckps have done.
   
       and ckp is also constrained by the minimal updates count restriction.

       but we can scan the catalog to collect block metadata that has not been checkpoint.

       we are trying to update storage usage info every 5 minutes.

    3. cost
        
        * time: 1) decode serval ckps; 2) join 3 system tables.
        * space: need extra 8B + 8B + 8B + 8B = 32B for each blk update record in ckp

    4. accurate
       
	we only consider blocks that are already flushed to the remote or ready to flush.
       

