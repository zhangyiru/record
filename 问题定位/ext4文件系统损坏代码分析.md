## 日志信息

报错日志时间顺序：

```c
//scsi接收到错误
[sdc] tag#2274 FAILED Result: hostbyte=DID_SOFT_ERROR driverbyte=DRIVER_OK
EXT4-fs warning (device sdc): ext4_end_bio:323: I/O error IO writing to inode xxx
Buffer I/O error on device sdc, logical block
JBD2: Detected IO errors while flushing file data

//scsi接收到错误，sde的话是重启后盘符漂移
[sde] tag#610 FAILED Result: hostbyte=DID_SOFT_ERROR driverbyte=DRIVER_OK
Aborting journal on device sde-8
EXT4-fs error (device sde): ext4_journal_check_start:61 comm git: Detected aborted journal
EXT4-fs (sde): Remounting filesystem read-only
```



<img src="file:///C:/Users/z00585918/AppData/Roaming/eSpace_Desktop/UserData/z00585918/imagefiles/originalImgfiles/962AB4EC-473D-43BB-B6EC-FD26CDFD9788.png" alt="img" style="zoom:150%;" />

![img](file:///C:/Users/z00585918/AppData/Roaming/eSpace_Desktop/UserData/z00585918/imagefiles/originalImgfiles/8A128E15-811E-48CC-B478-C0D048B63BC0.png)



## 流程梳理

```c
kjournald2 //[jbd2/dm-0-8]
	jbd2_journal_commit_transaction
		journal_finish_inode_data_buffers
    		err = journal_finish_inode_data_buffers(journal, commit_transaction);
				filemap_fdatawait_keep_errors
    				filemap_check_and_keep_errors
    					return AS_EIO; //IO error on async write
            if (err) {
                printk(KERN_WARNING
                    //"JBD2: Detected IO errors while flushing file data on %s\n", journal->j_devname);
                if (journal->j_flags & JBD2_ABORT_ON_SYNCDATA_ERR) //第一次没有走进来，只是打印了warn；
                    jbd2_journal_abort(journal, err);
                		__journal_abort_soft
                            __jbd2_journal_abort_hard
                                //Aborting journal on device
                err = 0;
            }
				

sys_write
	ksys_write
		__vfs_write
			new_sync_write
				call_write_iter
                	.write_iter = generic_file_write_iter

generic_file_write_iter
    __generic_file_write_iter
        generic_perform_write
            a_ops->write_begin
                ext4_da_write_begin
                    ext4_journal_start
                        __ext4_journal_start
                            __ext4_journal_start_sb	
                                ext4_journal_check_start
                                    ext4_abort(sb, "Detected aborted journal");
                       					__ext4_abort
                       						if (sb_rdonly(sb) == 0) {
												ext4_msg(sb, KERN_CRIT, "Remounting filesystem read-only");
                       				return -EROFS; /* Read-only file system */                    
```



## ext4错误处理

```
ext4_error
	__ext4_error	
		ext4_handle_error
			jbd2_journal_abort
```

