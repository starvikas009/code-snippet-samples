# Sample query
## Alter Table
#
### Create table
<details><summary>Click to expand!</summary>
    
>

    -- meg_cps.log_db_record_v2 definition
    CREATE TABLE `log_db_record_v2` (
        `id` int NOT NULL AUTO_INCREMENT,
        `epoch_sec` decimal(20,6) DEFAULT NULL,
        `logtime` datetime(6) DEFAULT NULL,
        `loglevel` varchar(50) DEFAULT NULL,
        `methodname` varchar(512) DEFAULT NULL,
        `linenum` int DEFAULT NULL,
        `thread` varchar(200) DEFAULT NULL,
        `msg` text,
        `req_id` varchar(200) DEFAULT NULL,
        `req_csn` varchar(200) DEFAULT NULL,
        `req_ts` int DEFAULT NULL,
        `req_guess` varchar(20) DEFAULT NULL,
        `billing` varchar(20) DEFAULT NULL,
        `logfile` varchar(500) DEFAULT NULL,
        PRIMARY KEY (`id`),
        KEY `log_db_record_v2_epoch_sec_IDX` (`epoch_sec`) USING BTREE,
        KEY `log_db_record_v2_logfile_IDX` (`logfile`,`thread`,`linenum`) USING BTREE
    ) ENGINE=InnoDB AUTO_INCREMENT=37117 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
    ;
    
</details>


#
### Create view 
<details><summary>Click to expand!</summary>

>
    CREATE OR REPLACE VIEW log_req as
    select t.logtime startime, t.*
    from meg_cps.meg_log t
    where t.req_id is not NULL and t.req_guess is NULL
    ;
    CREATE OR REPLACE VIEW REQ_SERVER_RES AS
    -- Complete Response from SERVER received for Request 
    -- Host: my.vm.com 
    -- Referer: https://abs.com
    --  Response Code [200], 
    -- Bytes Received [9978]
    SELECT 
        REPLACE(REGEXP_SUBSTR(msg , 'received for Request.{10}[^ ]*'), 'received for ', '') req_url,
        REPLACE(REGEXP_SUBSTR(msg , 'Host: [^ ]*,?'), 'received for ', '') host,
        REPLACE(REGEXP_SUBSTR(msg , 'Response Code[^,]*,?'), 'received for ', '') response_code,
        REPLACE(REGEXP_SUBSTR(msg , 'Bytes Received[^,]*,?'), 'received for ', '') bytes,
        REPLACE(REGEXP_SUBSTR(msg , 'id: [^, ]*,?'), 'received for ', '') reqhex,
        REPLACE(REGEXP_SUBSTR(msg , 'L:/[^, ]*,?'), 'received for ', '') req_l,
        REPLACE(REGEXP_SUBSTR(msg , 'R:[^, \]]*,?'), 'received for ', '') req_r,
        REPLACE(REGEXP_SUBSTR(msg , 'Referer:[^,]*,?'), 'received for ', '') referer,
	v.*
		FROM meg_cps.log_db_record_v2  v
		WHERE msg like '%Complete Response from SERVER received%';

</details>

#
### Insert rows from other table

<details><summary>Click to expand!</summary>

>
    INSERT INTO meg_cps.log_req 
    (meg_log_id, epoch_sec,logtime,loglevel,methodname,linenum,thread,msg,req_id,req_csn,req_ts,billing, logfile)
    select id,epoch_sec, logtime, loglevel, methodname,linenum,thread, msg,req_id,req_csn, req_ts,  billing,  logfile
    FROM meg_cps.log_db_record_v2 
    where req_id is NOT NULL and req_guess is NULL
    ;

</details>

#
### Add Index to a table
<details><summary>Click to expand!</summary>

    ALTER TABLE `log_db_record_v2` ADD INDEX `log_db_record_v2_epoch_sec_IDX` (`epoch_sec`) ;
    ALTER TABLE `log_db_record_v2` ADD INDEX `log_db_record_v2_logtime_IDX` (`logtime`);
    ALTER TABLE `log_db_record_v2` ADD INDEX `log_db_record_v2_file_thread_IDX` (`logfile`, `thread`);
    ALTER TABLE `log_db_record_v2` ADD INDEX `log_db_record_v2_req_IDX` (`req_id`);
    ALTER TABLE `log_db_record_v2` ADD INDEX `log_db_record_v2_req_ts_IDX` (`req_ts`, `req_id`);

    CREATE UNIQUE INDEX log_req_req_id_IDX USING BTREE ON meg_cps.log_req (req_id,req_ts,epoch_sec)
    ;
</details>

#

# Deduplicate
## Deduplicate: Step 1
### Check number of total rows and unique tuples
<details>
  <summary>Click to expand!</summary>
> 
    -- total count (762369)
    select count(*) from log_db_record_v2;

    -- total unique tuples (762369)
    WITH T_MIN_ID AS
    (
        SELECT  MIN(id) minid, count(*) cn, epoch_sec, `logtime` , loglevel, methodname, linenum, thread , msg
        FROM log_db_record_v2 
        GROUP BY epoch_sec, `logtime` , loglevel, methodname, linenum, thread , msg
        having cn > 0
    )
    SELECT count(*) from T_MIN_ID
    ;

    -- first row for each unique tuple
    WITH T_MIN_ID AS
    (
        SELECT  MIN(id) minid, epoch_sec, `logtime` , loglevel, methodname, linenum, thread , msg
        FROM log_db_record_v2 
        GROUP BY epoch_sec, `logtime` , loglevel, methodname, linenum, thread , msg
    )
    , T_ROW_MINID AS 
    (
        SELECT T2.* from T_MIN_ID 
            inner join log_db_record_v2 T2 on (T_MIN_ID.minid = T2.id)
    )
        SELECT T_ROW_MINID.* from T_ROW_MINID
    ;
</details>


## Deduplicate : Step 2
### Delete duplicate rows
<details>
  <summary>Click to expand!</summary>

  ### Find duplicate rows
  ### that are duplicate of one already existing row with lower ID
    -- second row onwards for each unique tuple
    WITH T_MIN_ID AS
    (
        SELECT  MIN(id) minid, epoch_sec, `logtime` , loglevel, methodname, linenum, thread , msg
        FROM log_db_record_v2 
        GROUP BY epoch_sec, `logtime` , loglevel, methodname, linenum, thread , msg
    )
    SELECT T2.* from T_MIN_ID T1, log_db_record_v2 T2 
        where 
            T1.epoch_sec = T2.epoch_sec
            and T1.loglevel = T2.loglevel 
            and T1.methodname = T2.methodname 
            and T1.linenum = T2.linenum 
            and T1.thread = T2.thread 
            and T1.msg = T2.msg 
            and T1.minid < T2.id
        ;

    -- second row onwards for each unique tuple (option B)
        SELECT T_DUPL_ROW.* FROM
        (
            SELECT  T2.* FROM log_db_record_v2 T2 ,
                    (
                        SELECT  MIN(id) minid, count(*) cn, epoch_sec, `logtime` , loglevel, methodname, linenum, thread , msg
                        FROM log_db_record_v2 
                        GROUP BY epoch_sec, `logtime` , loglevel, methodname, linenum, thread , msg
                    ) T1
                    where 
                        T1.cn > 1
                        and T1.epoch_sec = T2.epoch_sec
                        and T1.loglevel = T2.loglevel 
                        and T1.methodname = T2.methodname 
                        and T1.linenum = T2.linenum 
                        and T1.thread = T2.thread 
                        and T1.msg = T2.msg 
                        and T1.minid < T2.id
        )T_DUPL_ROW
        ;

  ### Mark duplicate rows for deletion
    -- all of these rows for deletion
    -- add the ID of each row that needs to be deleted
    INSERT INTO `idtable` (ext_id , ts, ext_table)
    SELECT T_DUPL_ROW.ID, "2021-04-16 15:24:35" ts, "log_db_record_v2-deletion" ext_table FROM
    (
        SELECT  T2.* FROM log_db_record_v2 T2 ,
                (
                    SELECT  MIN(id) minid, count(*) cn, epoch_sec, `logtime` , loglevel, methodname, linenum, thread , msg
                    FROM log_db_record_v2 
                    GROUP BY epoch_sec, `logtime` , loglevel, methodname, linenum, thread , msg
                ) T1
                where 
                    T1.cn > 1
                    and T1.epoch_sec = T2.epoch_sec
                    and T1.loglevel = T2.loglevel 
                    and T1.methodname = T2.methodname 
                    and T1.linenum = T2.linenum 
                    and T1.thread = T2.thread 
                    and T1.msg = T2.msg 
                    and T1.minid < T2.id
    ) T_DUPL_ROW
    ;

  ### Delete duplicate rows
    -- Delete rows marked for deletion
    DELETE from log_db_record_v2 
        where id in 
            (SELECT ext_id 
            from idtable 
            where ts = "2021-04-16 15:24:35" and ext_table="log_db_record_v2-deletion");
</details>

#

# Regex
1. Search using regex
    <details><summary>Click to expand!</summary>

    ### Search using regex
            SELECT *  FROM meg_cps.log_db_record v
            where msg REGEXP('\\W\\w+\\-\\d{7}\\W');

    ### Extract sub-string matching regex
            select REGEXP_SUBSTR(msg , '\\W\\w+\\-\\d{7}\\W') newreqid, v.*
            FROM meg_cps.log_db_record v
            where msg REGEXP('\\W\\w+\\-\\d{7}\\W');

    </details>

2. Replace part of string
    <details><summary>Click to expand!</summary>
    1. Update with replace
                UPDATE meg_cps.log_db_record_v2 v SET req_id = REPLACE(req_id , ']', '') WHERE id = 1;

    2. Select with Replace

            -- Another example selecting modified string
            SELECT REPLACE(REGEXP_SUBSTR(req_id , '\\-(\\d{7})$') , '-', '') ts0, REGEXP_SUBSTR(req_id , '\\-(\\d{7})$') ts1, req_id, v.*
            FROM meg_cps.log_db_record_v2 v
            where msg REGEXP('\\W\\w+\\-\\d{7}\\W') ;
        
    </details>

#




# --- ignore ---
### A collapsible section with markdown
<details>
  <summary>Click to expand!</summary>
  
  ## Heading
  1. A numbered
  2. list
     * With some
     * Sub bullets
</details>






### Example A collapsible section with markdown
<details><summary>SAMPLE</summary>

    My newline containing code
    here is another
        why

</details>
