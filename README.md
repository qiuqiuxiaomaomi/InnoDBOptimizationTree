# InnoDBOptimizationTree
Mysql InnoDB存储引擎性能调优

<pre>
update
      InnoDB内部的流程：
          1）将数据读入innodb_buffer_pool，并对相关记录加独占锁。
          2）将undo信息写入undo表空间的回滚段中。
          3）更改缓存页中的数据，并将更新记录写入redo buffer中；
          5）提交时，根据 innodb_flush_log_at_trx_commit的设置，用不同的方式将
             redo buffer中的更新记录刷新到innodb_redo_log_file中，然后释放独占锁。
          6）最后，后台线程根据需要择机将缓存中更新过的数据刷新到磁盘文件中。

      ---
	  LOG
	  ---
	  Log sequence number 1708635750      // 上次数据页的修改，还没有刷新到日志文件的lsn号
	  Log flushed up to   1708635750      // 上次成功操作，已经刷新到日志文件中的lsn号
	  Pages flushed up to 1708635750
	  Last checkpoint at  1708635741      // 上次检查点成功完成时的lsn号，以为着恢复的起点

      其中 lsn(log sequence number) 称为日志序列号,它实际上对应日志文件的偏移量，
      其生成公式是： 
                 新的 lsn = 旧的 lsn + 写入的日志大小
</pre>


<pre>
InnoDB Buffer Pool

          用户对数据库的最基本要求就是能高效的读取和存储数据，但是读写数据都涉及到与低速的设备交互，为了弥补两者之间的速度差异，
      所以有数据库都有缓存吃，用来管理相应的数据页，提高数据库的效率，当然也因为引入了这一中间层，数据库对内存的管理变得相对复杂。
</pre>

<pre>
InnoDB配置参数解析

Innodb_buffer_pool_dump_status	     Dumping of buffer pool not started
Innodb_buffer_pool_load_status	     Buffer pool(s) load completed at 181108 20:03:54
Innodb_buffer_pool_resize_status
###Innodb buffer pool缓存池中包含数据的页的数目，包括脏页，单位是page	
Innodb_buffer_pool_pages_data	     75825
Innodb_buffer_pool_bytes_data	     1242316800
###Innodb buffer pool缓存池中脏页的数目，单位是page
Innodb_buffer_pool_pages_dirty	     3304
Innodb_buffer_pool_bytes_dirty	     54132736
###Innodb buffer pool中刷新页请求的数目，单位是page
Innodb_buffer_pool_pages_flushed	 143095181
###Innodb buffer pool中剩余页数目，单位是page
Innodb_buffer_pool_pages_free	     708211
###Innodb buffer pool缓存池中当前已经被用作管理用途或hash index而不能用作为普通数据页的数目，单位是page
Innodb_buffer_pool_pages_misc	     2396
###Innodb buffer pool页总数目，单位是page
Innodb_buffer_pool_pages_total	     786432

Innodb_buffer_pool_read_ahead_rnd	 0
###后端预读线程读取到Innodb buffer pool的页的数目
Innodb_buffer_pool_read_ahead	     5315
###预读的页数，但是没有被读取就从缓冲池中被替换的页的数目，一般用来判断预读的效率
Innodb_buffer_pool_read_ahead_evicted	0
###进行逻辑读取时无法从缓冲池中获取而执行单页读取的次数。单位是次。
Innodb_buffer_pool_read_requests	   570169913484
###Innodb进行逻辑读的数量，单位是次
Innodb_buffer_pool_reads	         52823
###写入Innodb缓冲池通常在后台进行，但有必要在没有干净页的时候读取或者创建页，有必先等待页被刷新。Innodb的IO线程从数据文件中
###读取了数据要写入buffer pool的时候，需要等待空闲页的次数。
Innodb_buffer_pool_wait_free	     0
###写入InnoDB缓冲池的次数
Innodb_buffer_pool_write_requests	 2785929325

###Innodb进行fsync的次数
Innodb_data_fsyncs	                 13970945
###Innodb当前挂起fsyncs的次数
Innodb_data_pending_fsyncs	         0
###Innodb当前挂起的读操作数
Innodb_data_pending_reads	         0
###Innodb当前挂起的写操作次数
Innodb_data_pending_writes	         0
###Innodb读取的总数据量字节
Innodb_data_read	                 953750016
###Innodb数据读取总次数
Innodb_data_reads	                 58475
###Innodb数据写入总次数
Innodb_data_writes	                 152704758
###Innodb写入的总数据量字节
Innodb_data_written	                 2507431455232


###因日志缓存太小而必须等待其被写入所造成的等待次数
Innodb_log_waits	0
###Innodb日志写入请求数
Innodb_log_write_requests	3803295
###Innodb log buffer写入log file的总次数，
Innodb_log_writes	4827332

###Innodb log buffer进行fsync的总次数
Innodb_os_log_fsyncs	6043038
###当前挂起的fsync日志文件次数
Innodb_os_log_pending_fsyncs	0
###当前挂起的写log file的次数
Innodb_os_log_pending_writes	0
###写入日志文件的字节数
Innodb_os_log_written	5913782272


###编译的Innodb页大小
Innodb_page_size	16384
###InnoDB总共的页的数量
Innodb_pages_created	17620
###InnoDB总共读取的页的数量
Innodb_pages_read	58212
###InnoDB总共写入的页的数量
Innodb_pages_written	143192002

###InnoDB当前正在等待行锁的数量
Innodb_row_lock_current_waits	0
###InnoDB获取行锁的总消耗时间，ms
Innodb_row_lock_time	434911
###InnoDB获取行锁的平均等待时间，单位ms
Innodb_row_lock_time_avg	6
###InnoDB获取行锁的最大等待时间
Innodb_row_lock_time_max	88
###InnoDB等待获取行锁的次数
Innodb_row_lock_waits	69582
###InnoDB表中删除的行数
Innodb_rows_deleted	54010
###InnoDB表中插入的行数
Innodb_rows_inserted	264763433
###InnoDB表中读取的行数
Innodb_rows_read	2212269253959
###InnoDB表中更新的行数
Innodb_rows_updated	2165997
</pre>