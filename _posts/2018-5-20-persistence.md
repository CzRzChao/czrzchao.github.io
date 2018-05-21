---
title: redis源码解读(十一):数据持久化
date: 2018-5-20 18:11:30
categories: 
- redis
tags: 
- redis源码解析
- c
---

虽然 *redis* 是一个内存数据库，但也提供了数据持久化的解决方案。*redis* 的作者antirez大神对 *redis* 的持久化做了一个系统性的论述。在了解实现细节之前，建议先看看作者的论述。

> 原文：[Redis persistence demystified](http://oldblog.antirez.com/post/redis-persistence-demystified.html)  
> 译文：[解密Redis持久化](https://searchdatabase.techtarget.com.cn/7-19848/)



# 持久化方案
 *redis* 提供了两种持久化方案，分别是RDB和AOF。RDB是全量数据持久化，通过遍历所有数据库中的所有键值对，全量落地为2进制文件。AOF全称为append only file，是 *redis* 命令的增量记录。其他一些细节就不扯了，antirez大神的文章里都有，去找吧！

# RDB

## 触发方式
RDB有两种触发方式，首先是通过client命令手动触发，有SAVE和BGSAVE两种方式；还有一种是被动触发，通过配置一定条件，自动触发BGSAVE命令。

## 自动保存
先看自动保存的配置方式：在config文件中添加save配置，例如`save 900 10`，服务器在900s内对数据库进行了至少10次修改。redis支持多RDB配置，任意一个条件满足都会触发BGSAVE。  

`redisServer`持有一个`saveparam`的数组保存自动触发配置：

```c
struct saveparam {
    time_t seconds;
    int changes;
};

struct redisServer {
    // ...
    /* RDB persistence */
    long long dirty;                /* Changes to DB from the last save */  // db变更次数
    long long dirty_before_bgsave;  /* Used to restore dirty on failed BGSAVE */
    pid_t rdb_child_pid;            /* PID of RDB saving child */
    struct saveparam *saveparams;   /* Save points array for RDB */ // rdb save的配置
    int saveparamslen;              /* Number of saving points */
    char *rdb_filename;             /* Name of RDB file */
    int rdb_compression;            /* Use compression in RDB? */
    int rdb_checksum;               /* Use RDB checksum? */
    time_t lastsave;                /* Unix time of last successful save */ // 上一次执行save的时间点
    time_t lastbgsave_try;          /* Unix time of last attempted bgsave */
    time_t rdb_save_time_last;      /* Time used by last RDB save run. */
    time_t rdb_save_time_start;     /* Current RDB save start time. */
    int rdb_bgsave_scheduled;       /* BGSAVE when possible if true. */
    int rdb_child_type;             /* Type of save by active child. */
    int lastbgsave_status;          /* C_OK or C_ERR */
    int stop_writes_on_bgsave_err;  /* Don't allow writes if can't BGSAVE */
    int rdb_pipe_write_result_to_parent; /* RDB pipes used to return the state */
    int rdb_pipe_read_result_from_child; /* of each slave in diskless SYNC. */
    // ...
}
```

之前的文章中有提到 *redis* 的事件循环中有定期执行的时间事件，如果没有正在执行的bgsave或aof rewrite，就会对`saveparams`中所有的配置进行检测，是否需要进行BGSAVE。  

```c
int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData) { // redis的定时任务 系统默认为每秒跑10次
    // ...
    /* Check if a background saving or AOF rewrite in progress terminated. */
    if (server.rdb_child_pid != -1 || server.aof_child_pid != -1 ||
        ldbPendingChildren())
    {   // 如果有bgsave或者aof子进程
        // ...
        }
    } else {
         for (j = 0; j < server.saveparamslen; j++) {
            struct saveparam *sp = server.saveparams+j;
            // 校验是否满足rdb的saveparam的触发条件
            if (server.dirty >= sp->changes &&
                server.unixtime-server.lastsave > sp->seconds &&
                (server.unixtime-server.lastbgsave_try >
                 CONFIG_BGSAVE_RETRY_DELAY ||
                 server.lastbgsave_status == C_OK))
            {   
                serverLog(LL_NOTICE,"%d changes in %d seconds. Saving...",
                    sp->changes, (int)sp->seconds);
                rdbSaveBackground(server.rdb_filename);  // 调用BGSAVE
                break;
            }
         }
         // ...
    }
    // ...
}
```
`rdbSaveBackground`就是`BGSAVE`命令调用的函数，该函数会fork一个子进程执行SAVE操作，使服务主进程不被阻塞。并且由于linux `copy-on-write`的特性，正常情况下不会出现内存使用翻倍的情况。

```c
int rdbSaveBackground(char *filename) { // 子进程保存 非阻塞
    pid_t childpid;
    long long start;

    if (server.aof_child_pid != -1 || server.rdb_child_pid != -1) return C_ERR;

    server.dirty_before_bgsave = server.dirty;
    server.lastbgsave_try = time(NULL);

    start = ustime();
    if ((childpid = fork()) == 0) {
        int retval;

        /* Child */
        closeListeningSockets(0);   // 关闭子进程的监听
        redisSetProcTitle("redis-rdb-bgsave");
        retval = rdbSave(filename); // 调用rdbsave保存rdb文件
        if (retval == C_OK) {
            size_t private_dirty = zmalloc_get_private_dirty();

            if (private_dirty) {
                serverLog(LL_NOTICE,
                    "RDB: %zu MB of memory used by copy-on-write",
                    private_dirty/(1024*1024));
            }
        }
        exitFromChild((retval == C_OK) ? 0 : 1);    // 调用_exit退出
    } else { // 主进程进行BGSAVE状态记录
        /* Parent */
        server.stat_fork_time = ustime()-start;
        server.stat_fork_rate = (double) zmalloc_used_memory() * 1000000 / server.stat_fork_time / (1024*1024*1024); /* GB per second. */   // fork速度
        latencyAddSampleIfNeeded("fork",server.stat_fork_time/1000);
        if (childpid == -1) {   // fork失败
            server.lastbgsave_status = C_ERR;
            serverLog(LL_WARNING,"Can't save in background: fork: %s",
                strerror(errno));
            return C_ERR;
        }
        serverLog(LL_NOTICE,"Background saving started by pid %d",childpid);
        server.rdb_save_time_start = time(NULL);
        server.rdb_child_pid = childpid;
        server.rdb_child_type = RDB_CHILD_TYPE_DISK;
        updateDictResizePolicy();
        return C_OK;
    }
    return C_OK; /* unreached */
}
```
有个小细节，在退出子进程的时候，*redis* 采用的是`_exit`而不是`exit`，因为父进程可能对文件对象进行操作，`exit`会对清除IO缓存，可能会父进程造成影响。

# save
殊途同归，不论是自动触发还是SAVE和BGSAVE，最终都会走到`rdbSave`函数：

```c
int rdbSave(char *filename) {   // 将db中的数据保存到rdb文件中
    char tmpfile[256];
    char cwd[MAXPATHLEN]; /* Current working dir path for error messages. */
    FILE *fp;
    rio rdb;
    int error = 0;

    snprintf(tmpfile,256,"temp-%d.rdb", (int) getpid());    // 创建一个temp文件
    fp = fopen(tmpfile,"w");
    if (!fp) {
        char *cwdp = getcwd(cwd,MAXPATHLEN);
        serverLog(LL_WARNING,
            "Failed opening the RDB file %s (in server root dir %s) "
            "for saving: %s",
            filename,
            cwdp ? cwdp : "unknown",
            strerror(errno));
        return C_ERR;
    }

    rioInitWithFile(&rdb,fp);   // 将rio初始化为文件类型
    if (rdbSaveRio(&rdb,&error) == C_ERR) { // 保存rdb文件
        errno = error;
        goto werr;
    }

    /* Make sure data will not remain on the OS's output buffers */
    if (fflush(fp) == EOF) goto werr;
    if (fsync(fileno(fp)) == -1) goto werr;
    if (fclose(fp) == EOF) goto werr;

    if (rename(tmpfile,filename) == -1) {   // 原子操作 重命名
        char *cwdp = getcwd(cwd,MAXPATHLEN);
        serverLog(LL_WARNING,
            "Error moving temp DB file %s on the final "
            "destination %s (in server root dir %s): %s",
            tmpfile,
            filename,
            cwdp ? cwdp : "unknown",
            strerror(errno));
        unlink(tmpfile);
        return C_ERR;
    }

    serverLog(LL_NOTICE,"DB saved on disk");
    server.dirty = 0;
    server.lastsave = time(NULL);
    server.lastbgsave_status = C_OK;
    return C_OK;

werr:
    serverLog(LL_WARNING,"Write error saving DB on disk: %s", strerror(errno));
    fclose(fp);
    unlink(tmpfile);
    return C_ERR;
}
```

上述代码大部分为流程分支处理，要点有二：

1. 通过先创建临时文件，写入后再原子性的rename，确保rdb文件都是完整可用的
2. 出现了一个叫做rio的数据类型，并且初始化为file类型

`rio`是 *redis* 的io封装，所有socket、file、buffer的io都封装在`rio`中，rdb就是将rio初始化为file类型，进行文件的读写操作。除了`rio`，*redis* 还有对后台io操作封装的`bio`。


# AOF