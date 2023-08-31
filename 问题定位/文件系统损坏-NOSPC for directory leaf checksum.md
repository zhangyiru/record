## 背景

文件系统损坏，报错No space for directory leaf checksum


## debugfs

```shell
dump_extents, extents, ex
---Dump extents information
```

![image](https://github.com/zhangyiru/record/assets/15630061/c7da8121-97fb-49fc-b4ac-c338b5d71b6c)


```shell
block_dump, bdump, bd
---Dump contents of a block
```


```c
struct ext4_dir_entry_tail {
	__le32	det_reserved_zero1;	/* Pretend to be unused */
	__le16	det_rec_len;		/* 12 */
	__u8	det_reserved_zero2;	/* Zero name length */
	__u8	det_reserved_ft;	/* 0xDE, fake file type */
	__le32	det_checksum;		/* crc32c(uuid+inum+dirblock) */
};

//初始化dir entry tail
/* checksumming functions */
void initialize_dirent_tail(struct ext4_dir_entry_tail *t,
			    unsigned int blocksize)
{
	memset(t, 0, sizeof(struct ext4_dir_entry_tail));
	t->det_rec_len = ext4_rec_len_to_disk(
			sizeof(struct ext4_dir_entry_tail), blocksize);
	t->det_reserved_ft = EXT4_FT_DIR_CSUM;
}

ext4_find_entry
	ext4_dirent_csum_verify
		t = get_dirent_tail(inode, dirent);
			//ext4_dir_entry_tail结构体下的数据值都不对
			if (t->det_reserved_zero1 || le16_to_cpu(t->det_rec_len) != sizeof(struct ext4_dir_entry_tail) || t->det_reserved_zero2 || t->det_reserved_ft != EXT4_FT_DIR_CSUM)
				return NULL;
		if (!t)
			warn_no_space_for_csum
```



```shell
ncheck
---Do inode->name translation

ncheck 59
```

```shell
show_super_stats, stats  
---Show superblock statistics
```


logdump -aOS  /root/jbd.log


```shell
icheck                  
---Do block->inode translation

icheck 8748 通过block查看inode
```

## 结论

通过debugfs查看对应报错inode对应block，发现数据被踩了，需要排查是否存在写裸盘的操作
