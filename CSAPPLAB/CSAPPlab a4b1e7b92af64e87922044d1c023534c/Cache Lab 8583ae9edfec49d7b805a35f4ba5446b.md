# Cache Lab

编写一个高速缓存模拟器以及要求优化矩阵转置的核心函数，以最小化对模拟的高速缓存的不命中次数。

- 缓存机制
    
    CPU在执行时，需要的指令和数据通过内存总线和系统总线由内存传送到寄存器，再由寄存器送入ALU，SRAM的速度介于DRAM(主存)和CPU之间，我们把接下来可能会用到的数据放在SRAM(Cache)中，当CPU需要数据时先去查Cache，如果Cache中有(hit)，就不用再去访问主存了。
    
    - Cache结构
        
        ![Untitled](Cache%20Lab%208583ae9edfec49d7b805a35f4ba5446b/Untitled.png)
        
    - 命中结果
        
        ![Untitled](Cache%20Lab%208583ae9edfec49d7b805a35f4ba5446b/Untitled%201.png)
        
        如果我们的程序请求一个数据字，这个数据字存储在编号为10的块中，将分以下几种情况考虑：
        
        - 冷不命中：高速缓存中为空
        - 缓存不命中：高速缓存中有数据块，但没有数据块10
        - **冲突不命中**：高速缓存中有数据，将内存中的数据块放置到高速缓存中时，发生了冲突
        - 缓存命中：高速缓存中有数据块10，直接返回CPU
        
        当一条加载指令指示CPU从主存地址A中读取一个字w时，会将该主存地址A发送到高速缓存中，则高速缓存会根据以下步骤判断地址A是否命中：
        
        - 组选择：根据地址划分，将中间的 **s位** 表示为无符号数作为组的索引，可得到该地址对应的组
        - 行匹配：根据地址划分，可得到 **t位** 的标记位（由于组内的任意一行都可以包含任意映射到该组的数据块，所以就要线性搜索组中的每一行），判断是否有和标志位匹配且设置了有效位的行 ，如果存在，则缓存命中，否则缓冲不命中
        - 字选择：如果找到了对应的高速缓存行，则可以将 **b位** 表示为无符号数作为块偏移量 ，得到对应位置的字
        
        当高速缓存命中时：会很快抽取出字w，并将其返回给CPU
        
        如果缓存不命中时：CPU会进行等待，高速缓存会向主存请求包含字w的数据块，当请求的块从主存到达时，高速缓存会将这个块保存到它的一个高速缓存行中，然后从被存储的块中抽取出字w，将其返回给CPU
        
    - Cache映射
        
        （S，E，B，m）=（4，1，2，4）
        
        假设某个Cache有：4个组，每个组1行（cacheline），每个块2字节，地址m为4位：
        
        ![Untitled](Cache%20Lab%208583ae9edfec49d7b805a35f4ba5446b/Untitled%202.png)
        
        因为地址m为4位，所以地址空间可以分为16个区域，编号为 地址0 ~ 地址15
        
        每个块的大小为2字节，所以两个内存区域可以组成一个块，例如块0是由地址0和地址1组成块1是由地址2和地址3组成的，通过 **标记位（Tag） 和 索引位（Index）可以唯一确定每一个块**，共有8个块(编号7)
        
        但是 Cache 只有4个组set（每个组只有1行），所以会有两个块被映射到同一个组的情况，比如：块0（地址0和地址1）和块4（地址8和地址9）被都被映射到了set0，而块1和块5都被映射到了set1
        
        也就是说，块和组可能不是一一对应的，这就导致了冲突不命中
        
        // 为什么不采用高位来作为组索引位（s位）？
        
        - 直接映射高速缓存
            
            根据 cache 每个组中行数的不同，cache 被分为不同的类型，当行数为 1 时（**E==1**），这种 cache 被称为直接映射。
            
            假设有 S 组，每组由1行组成，缓存块为8字节
            
            组选择：
            
            ![Untitled](Cache%20Lab%208583ae9edfec49d7b805a35f4ba5446b/Untitled%203.png)
            
            CPU发出地址（就是右边那个）要取数据字，高速缓存将该地址分解为三部分：标记位，组索引，块偏移
            
            上图的 组索引（s位）为 “0x1” ，所以索引第2个组（0开始）。
            
            行匹配：
            
            ![Untitled](Cache%20Lab%208583ae9edfec49d7b805a35f4ba5446b/Untitled%204.png)
            
            检查地址中的 标记位 与缓存行中的 标记位 是否匹配：
            
            - 如果匹配，将进行下一步字选择
            - 如果不匹配，则表示未命中（高速缓存必须从内存中重新取数据块，在行中覆盖此块）
            
            字选择：
            
            ![Untitled](Cache%20Lab%208583ae9edfec49d7b805a35f4ba5446b/Untitled%205.png)
            
            块偏移（b位）为 “0x4” ，所以索引标记为“0x4”的块，返回给CPU
            
             // 字选择的操作是目标数据块的起始地址
            
        - 局部性原理
            - 时间局部性：最近访问的数据可能在不久的将来会再次访问
            - 空间局部性：位置相近的数据常常在相近的时间内被访问
            
            根据局部性原理，我们可以把计算机存储系统组织成一个存储山，越靠近山顶，速度越快，但造价越昂贵，越靠近山底，速度越越慢，但造价越便宜
            
            上一层作为下一层的缓冲，保存下一层中的一部分数据
            
            ![Untitled](Cache%20Lab%208583ae9edfec49d7b805a35f4ba5446b/Untitled%206.png)
            
            - 分块技术可以提高时间局部性，使得每一行Cache被充分地使用：
            
            ```c
            int A[N][M];
            int B[M][N];
            int tmp;
            for (i = 0; i < N; i++)
                for (j = 0; j < M; j++) {
                    tmp = A[i][j];
                    B[j][i] = tmp;
                }
            ```
            
            假设Cache每行有32字节，每组1行，那么内存第一次循环时，会把 A[0] [0] ~ A[0] [7] 放入第1组，把 B[0] [0] ~ B[0] [7] 放入第2组。
            
            当第二次循环获取 A[0] [1] 时可以命中，但获取 B[1] [0] 时不会命中，并且把 B[1] [0] ~ B[1] [7] 放入第3组。
            
            以此类推，当Cache空间不够时，就可能覆盖前面的内容，导致数组B永远也不可能命中。但是如果可以把 N * M 的大矩阵分为小矩阵，使其可以存储下当前小矩阵中所有的 数组B 的值，就可以在多次不命中后再次命中。
            
- 替换策略
    
    当 CPU 要处理的字不在组中任何一行，且组中没有一个空行，那就必须从里面选取一个非空行进行替换。选取哪个空行进行替换呢？
    
    - LFU，最不常使用策略。替换在过去某个窗口时间内**引用次数最少**的那一行
    - LRU（**Least Recently Used**），最近最少使用策略。替换最后一次访问**时间最久远**的哪一行
        
        实现 ：S 个双向链表，每个链表有 E 个结点，对应于组中的每一行，每当访问了其中的一行，就把这个结点移动到链表的头部，后续替换的时候只需要选择链尾的结点就好了。但是可以使用相对简单的设置**时间戳最小的替换。**
        的办法
        
- CacheSimulator
    
    ```c
    #include "cachelab.h"
    #include <stdio.h>
    #include <stdlib.h>
    #include <unistd.h>
    #include <string.h>
    #include <math.h>
    #include <errno.h>
    #include <getopt.h>
    
    int b = 0, s = 0, B = 0, S = 0;
    int E = 0, verbosity = 0;
    // LRU从1开始，0为空行
    long long int lru_count = 1;
    char *trace_file = NULL; //?
    int hit_count = 0, miss_count = 0, eviction_count = 0;
    
    typedef long long unsigned mem_addr_t;
    struct cache_line_t // 行
    {
        mem_addr_t tag;
        int valid;
        unsigned int lru;
    };
    typedef struct cache_line_t *cache_set_t; // 组号
    typedef cache_set_t *cache_t;             // 二维数组cache
    
    cache_t cache;
    mem_addr_t set_index_mask = 0; // 组选择？
    
    void printUsage(char *name)
    {
        printf("Usage: %s [-hv] -s <num> -E <num> -b <num> -t <file>\n", name);
        puts("Options:");
        puts("  -h         Print this help message.");
        puts("  -v         Optional verbose flag.");
        puts("  -s <num>   Number of set index bits.");
        puts("  -E <num>   Number of lines per set.");
        puts("  -b <num>   Number of block offset bits.");
        puts("  -t <file>  Trace file.");
        puts("\nExamples:");
        printf("  linux>  %s -s 4 -E 1 -b 4 -t traces/yi.trace\n", name);
        printf("  linux>  %s -v -s 8 -E 2 -b 4 -t traces/yi.trace\n", name);
        exit(0);
    }
    
    void initCache()
    {
        cache = (cache_t)malloc(sizeof(cache_set_t) * S);
        for (int i = 0; i < S; i++)
        {
            cache[i] = (cache_set_t)malloc(sizeof(struct cache_line_t) * E);
            for (int j = 0; j < E; j++)
            {
                cache[i][j].lru = 0;
                cache[i][j].tag = 0;
                cache[i][j].valid = 0;
            }
        }
        set_index_mask = pow(2, s) - 1; // 2的s次方个组？
    }
    
    void acessData(mem_addr_t addr)
    {
        int eviction_line = 0;          // 要驱逐的行
        unsigned int eviction_lru = -1; // why
        mem_addr_t tag = addr >> (s + b);
        cache_set_t cache_set = cache[(addr >> b) & set_index_mask]; // 组选择 but why &set_index_mask?
    
        for (int i = 0;; i++)
        {               // 行匹配
            if (i >= E) // 遍历完所有行号仍找不到就未命中
            {
                ++miss_count;
                if (verbosity)
                    printf("miss ");
                // 在最后一组cache_line中选出时间戳最小的进行驱逐（替换）
                for (int ia = 0; ia < E; ++ia)
                {
                    if (cache_set[ia].lru < eviction_lru)
                    { // 找最小时间戳
                        eviction_line = ia;
                        eviction_lru = cache_set[ia].lru;
                    }
                }
    
                if (cache_set[eviction_line].valid)
                {
                    // 要替换的行是valid的，即该行是一条之前读入的数据而不是空行
                    ++eviction_count;
                    if (verbosity)
                        printf("eviction ");
                }
    
                // 模拟读入并覆盖数据到这个刚刚被删除（或本来就是空行）的cacheline里
                cache_set[eviction_line].valid = 1;
                cache_set[eviction_line].tag = tag;
                cache_set[eviction_line].lru = lru_count++;
                return;
            }
            if (cache_set[i].tag == tag && cache[i].valid)
                break; // 直接命中该行，跳出循环
        }
    
        hit_count++; // 直接命中行
        if (verbosity)
            printf("hit ");
        cache_set[i].lru = lru_count++
    }
    
    void replayTrace(char *trace_fn)
    {
        FILE *trace_fp = fopen(trace_fn, "r");
        if (!trace_fp)
        {
            int *err_num = __errno_location();
            char *err_str = strerror(*err_num);
            fprintf(stderr, "%s: %s\n", trace_fn, err_str);
            exit(1);
        }
        char buf[1000];
        while (fgets(buf, 1000, trace_fp))
        {
            unsigned int len = 0;
            mem_addr_t addr = 0;
            if (buf[1] == 'S' || buf[1] == 'L' || buf[1] == 'M')
            {
                sscanf(&buf[3], "%llx,%u", &addr, &len);
                if (verbosity)
                    printf("%c %llx,%u ", buf[1], addr, len);
                // 读取/写入数据
                // 写入数据同样需要判断数据是否存在与cache。如果数据不在，同样要将其读回cache
                accessData(addr);
                // 如果当前指令是修改指令，则上一条accessData读取数据，下一条的accessData写入数据
                if (buf[1] == 'M')
                    accessData(addr);
                if (verbosity)
                    putchar('\n');
            }
        }
        fclose(trace_fp);
    }
    
    void freeCache()
    {
        for (int i = 0; i < S; ++i )
            free(cache[i]);
        free(cache);
    }
    int main(int argc, char* argv[])
    {
        char c;
        while((c = getopt(argc, argv, "s:E:b:t:vh")) != -1)
        {
            switch ( c )
            {
            case 'E':
                E = atoi(optarg);
                break;
            case 'b':
                b = atoi(optarg);
                break;
            case 'h':
                printUsage(argv[0]);
                return 0;
            case 's':
                s = atoi(optarg);
                break;
            case 't':
                trace_file = optarg;
                break;
            case 'v':
                verbosity = 1;
                break;
            default:
                printUsage(argv[0]);
                return 0;
            }
        }
        if ( !s || !E || !b || !trace_file )
        {
            printf("%s: Missing required command line argument\n", *argv);
            printUsage(argv[0]);
        }
        S = pow(2, s);
        B = pow(2, b);
        initCache();
        replayTrace(trace_file);
        freeCache();
        printSummary(hit_count, miss_count, eviction_count);
        return 0;
    }
    ```