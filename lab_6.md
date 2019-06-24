# 在Ubuntu下编写程序,用信号量解决生产者——消费者问题

> 编写`pc.c`。

强烈推荐看看《APUE》这本书。信号量的使用书中都有介绍。具体实现代码如下：

```c
#include <stdio.h>
#include <semaphore.h>
#include <unistd.h>
#include <fcntl.h>
#include <stdlib.h>
#include <sys/wait.h>

#define BUFSIZE 10  /* 缓冲区大小，按照指导书要求设置为10 */

#define CONSUMER_NUM 4  /* 消费者进程数 */

#define MAX_NUM 500

int fd;
sem_t *empty; /* 空槽位的数量 */
sem_t *full; /* 满槽位的数量 */
sem_t *mutex;  /* 控制对文件互斥的访问 */

/**
 * 生产者在文件的最后添加数字
 */
void producer()
{
    int item_num = 0;
    while (item_num < MAX_NUM)
    {
        sem_wait(empty);
        sem_wait(mutex);
        if (lseek(fd, 0, SEEK_END) < 0)
            fprintf(stderr, "Error in producer lseek\n");
        write(fd, &item_num, sizeof(int));
        fsync(fd);
        sem_post(mutex);
        sem_post(full);
        ++item_num;
    }
    close(fd);
}

/**
 * 消费者在文件的开头读取数字，
 * 并且将读取过的数字删除（通过将后面的数字往前移实现）
 */
void consumer()
{
    int item;
    int file_len;
    int tmp_value;
    int j;

    do
    {
        sem_wait(full);
        sem_wait(mutex);
        if (lseek(fd, 0, SEEK_SET) < 0)
            fprintf(stderr, "Error in consumer lseek\n");

        if (read(fd, &item, sizeof(int)) == 0)
        {
            sem_post(mutex);
            sem_post(empty);
            break;
        }

        printf("%d: %d\n", getpid(), item);

        file_len = lseek(fd, 0, SEEK_END); 
        if (file_len < 0)
            fprintf(stderr, "Error when get file length\n");
        /* 通过将后面的数字前移来删除已经读取的数字 */
        /* 这种方式速度特别慢,不知道还有没有别的好办法 */
        for(j = 1; j < (file_len / sizeof(int)); j++) 
        { 
            lseek(fd, j * sizeof(int), SEEK_SET); 
            read(fd, &tmp_value, sizeof(int)); 
            lseek(fd, (j - 1) * sizeof(int), SEEK_SET); 
            write(fd, &tmp_value, sizeof(int)); 
        } 
        ftruncate(fd, file_len - sizeof(int));

        sem_post(mutex);
        sem_post(empty);
    }while(item < MAX_NUM - 1);

    sem_post(full);  /* 当第一个进程退出时通知另外一个阻塞在full上的进程,不然另一个进程永远不会退出了 */

    close(fd);
}

int main()
{
    char empty_name[64];
    char full_name[64];
    char mutex_name[64];
    int i;
    pid_t p_pid;  /* 生产者进程pid */

    /* 确保每个信号量有不同的名字 */
    /* from APUE */
    snprintf(empty_name, 64, "/%ld_empty", (long)getpid());    
    snprintf(full_name, 64, "/%ld_full", (long)getpid());
    snprintf(mutex_name, 64, "/%ld_mutex", (long)getpid());

    fd = open("share.file", O_CREAT | O_RDWR | O_TRUNC, 0666);

    empty = sem_open(empty_name, O_CREAT | O_EXCL, S_IRWXU, BUFSIZE);
    if (empty == SEM_FAILED)
    {
        fprintf(stderr, "Error when create empty\n");
        exit(0);
    }
    full = sem_open(full_name, O_CREAT | O_EXCL, S_IRWXU, 0);
    if (full == SEM_FAILED)
    {
        fprintf(stderr, "Error when create full\n");
        exit(0);
    }
    mutex = sem_open(mutex_name, O_CREAT | O_EXCL, S_IRWXU, 1);
    if (mutex == SEM_FAILED)
    {
        fprintf(stderr, "Error when create mutex\n");
        exit(0);
    }

    printf("Create semphore OK!\n");

    /* 消费者进程 */
    for (i = 0; i < CONSUMER_NUM; ++i)
    {
        if (!fork())
        {
            consumer();
            exit(0);
        }
    }

    /* 生产者进程 */
    if (!fork())
    {
        producer();
        exit(0);
    }
    /* 等待所有子进程结束 */
    while (waitpid(-1, NULL, 0) > 0)
        ;

    sem_close(empty);
    sem_close(full);
    sem_close(mutex);
    close(fd);
    
    return 0;
}
```

注意点在注释中都有体现。

# 在0.11中实现信号量,用生产者—消费者程序检验之

# 用信号量解决生产者—消费者问题

