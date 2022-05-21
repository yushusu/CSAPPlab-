# Malloc Lab

![Untitled](Malloc%20Lab%2084264ff30cef40a0947a0bce7b70e749/Untitled.png)

对每个进程，内核维护一个变量brk指向堆顶部，

动态内存分配器（mm?）把堆视为不同大小块的集合体，每个块就是一个连续的chunk，有free和allcated两种状态，

动态内存分配器分两种风格：显式释放已分配块/主动释放/只有调用者能释放掉chunk/mallocfree/newdelete；隐式释放已分配的块/自动释放/分配器自动检测不被利用的chunk/java、ml、lisp、垃圾回收机制

已分配块结构：

![Untitled](Malloc%20Lab%2084264ff30cef40a0947a0bce7b70e749/Untitled%201.png)

空闲块结构：

![Untitled](Malloc%20Lab%2084264ff30cef40a0947a0bce7b70e749/Untitled%202.png)

内存管理分为：

- 空闲块的组织方式
    - 隐式空闲链表(implicit free list)
        
        任何分配器都需要数据结构实现，来区别块边界、已分配块、空闲块，大部分分配器将这些信息嵌入块本身（块头部）。把堆按地址顺序组织成一个连续的，包含allocated chunk和free chunk的链表，头部flag标记是否空闲，这样做的缺点是申请新chunk的时**需要遍历整个链表**。
        
    - 显式空闲链表(explicit free list)
        
        块分配与堆块的总数呈线性关系，所以对于通用的分配器是不适用隐式空闲链表的，所以要将空闲块以某种形式组织起来，通过“free chunk->FD”和“free chunk->BK”把free chunk连接为一个双向链表。
        
    - 分离空闲链表(segregated free list)
- 空闲块的再申请（搜索合适的空闲块）/放置策略（placement policy）
    - 首次适配（first-fit）
        
        物理内存页管理器顺着双向链表（存空闲块的）搜索空闲内存区域，直到找到一个足够大的空闲区域，然后立刻将其进行分配（或分割后分配），此后的内存块都不做处理。这样做虽然搜索迅速但缺点式分割后剩下的内存块会越来越难利用。
        
        ```c
        static block_t *first_fit(size){
        		block_t *block;
        		for(block = heap_start; get_size(block) > 0; block = find_next(block)){
        				if(!(get_alloc(block)) && (asize <= get_size(block)))
        						return block;
        		}
        		return NULL;
        }
        ```
        
    - 下次适配（next-fit）
        
        类似首次适配，但不是每次都从链表的起始处开始搜索，而是从上一次查询结束的地方开始。
        
    - 最佳适配（best-fit）
        
        每次都遍历所有内存块，获取最合适的块，速度缓慢但剩下的内存块利用率高。
        
        ```c
        static block_t *best_fit(size_t asize){
        		block_t *block;
        		for(block = heap_start; get_size(block) > 0; block = find_next(block))
        				if(!(get_alloc(block)) && (asize == get_size(block)))
        						return block;
        		return NULL;
        }
        ```
        
    
- 空闲块的分割
    
    难点：碎片化管理
    
- 空闲块的合并
    
    > 边界标记（boundary tag），允许在常数时间内进行对前面块的合并。在每个块的结尾处添加一个脚部（footer，边界标记，原本内存块的头部就可以标记chunk的状态，脚部只是头部的一个副本），检査脚部判断前面一个块的起始位置和状态，这个脚部总是在当前块开始位置前一个字的距离。
    > 
    
    考虑当分配器**释放当前块时**所有可能存在的情况：
    
    - 前后块都已分配，不合并
    - 前分配，后空闲，NEXT_BLKP→SIZE + SIZE，更新脚部
    - 前空闲，后分配，PREV_BLKP→SIZE + SIZE，更新头部
    - 前后块都空闲，SIZE同时加上前后两块的SIZE，更新头部&脚部

练习：

- 隐式空闲链表+首次适配
    - 先用“extend_heap”初始化一个大小为CHUNKSIZE的大内存块，使其为free状态
    - 后续的malloc都在这个 大内存块中分割
    - 如果大内存块不够用，就再次调用“extend_heap”进行申请
    
- 隐式空闲链表+下次适配
    - 当下次适配遍历完所有的空闲块时，需要把BP重置为heap_listp（第一个内存块）
    - 只需要“mm_malloc”中记录一下“pre_listp”，重复记录可能会报错
    - 可以把“pre_listp”初始化为第二个内存块的BP（比如本程序），也可以初始化为第一个
- 显式空闲链表+首次适配

- mm_malloc
    
    功能：首先查找各个链表中是否存在所需要的chunk。
    
    - 如果有，则断开链表、切割chunk、设置chunk的头部与尾部（只有切割下的chunk会设置尾部）、将剩余chunk放回链表、返回用户地址一条龙。
    - 如果没有，则判断`top chunk`的空间是否足够分配(地址最高、位于堆顶的的free chunk，称为top chunk，该 chunk不属于任何bin，只由arena直接管理，当所有free chunk都无法满足用户需求时，如果top chunk够大，则top chunk将会被切割为两部分，第一部分分配给用户，第二部分会成为新的top chunk)
    - 如果足够，切割`top chunk`、设置chunk的首尾、重新设置`top_chunk`、将目标chunk的用户地址返回。
    - 如果不够，将当前`top chunk`插入链表中，重新分配一块超大内存给`top_chunk`指针，之后重新递归执行`mm_malloc`，返回该递归执行的返回值。
    
    检查：
    
    - 每个释放请求必须对应一个当前已分配的块，这个块是由一个以前的分配请求获得到的
    - 立刻响应请求
    - 不修改已经分配的chunk（申请的chunk不重叠）
    
    ```c
    //最好将所有指针强制转换为char*或void*类型再操作。
    void *mm_malloc(size_t size)
    {
        void* targetMemPtr = NULL;
        int targetChunksize = request2chunksize(size); // 对齐
        // 查找链表中是否存在需要的chunk
        for(int listIdx = getListIndx(targetChunksize); listIdx < HEAP_LIST_NUM; listIdx++)
        {
            // 如果当前遍历到的链表非空，并且该链条中最大的chunk不小于所申请空间的大小
            if(heap_listp[listIdx][0] != FD2HD(&heap_listp[listIdx][0]) 
                && GET_SIZE(heap_listp[listIdx][1]) >= targetChunksize)
            {
                // 遍历向下查找
                void* ptr = heap_listp[listIdx][0];
                while(GET_SIZE(ptr) < targetChunksize)
                    ptr = FD(ptr);
                // 此时已经获取到了所需要的chunk，但需要判断是否分割处理
                int current_chunkSize = GET_SIZE(ptr);
    
                int remainSize = current_chunkSize - targetChunksize;
                // 先断开链表   
                unlink_chunk(ptr);
                if(remainSize >= MIN_CHUNKSIZE)
                {
                    void* remainChunk = splitChunk(ptr, targetChunksize);
                    insertChunk2List(remainChunk);
                }
                else
                {
                    // 设置目标chunk的header
                    SET_SIZE(ptr, current_chunkSize);
                    SET_ALLOC(ptr);
                    SET_PREV_ALLOC(NEXT_CHUNK(ptr));
                }
                // 返回用户空间
                DBG(GET_ALLOC(ptr) && "mm_malloc: bad alloc status");
                targetMemPtr = HD2MEM(ptr);
                return targetMemPtr;
            }
        }
    
        // 如果所有的链表中的chunk都不满足，则需要向top chunk申请
        DBG(top_chunk && "mm_malloc： top_chunk不可能为空");
        // 必须保证top_chunk在任何情况下都有空间
        if(GET_SIZE(top_chunk) >= targetChunksize + MIN_CHUNKSIZE)
        {
            void* ptr = top_chunk;
            top_chunk = splitChunk(ptr, targetChunksize);
            DBG(GET_ALLOC(ptr) && "mm_malloc: bad alloc status");
            return HD2MEM(ptr);
        }
    
        // 如果所有的chunk都不满足，则需要向brk申请
        int extend_size = (targetChunksize + MIN_CHUNKSIZE > CHUNKSIZE ? 
            targetChunksize + MIN_CHUNKSIZE : CHUNKSIZE);
        void* extend_ptr = extend_heap(extend_size / WSIZE);
        // 如果没内存了，则返回空
        if(!extend_ptr)
            return NULL;
        SET_SIZE(extend_ptr, extend_size);
        SET_FREE(extend_ptr);
        SET_PREV_ALLOC(extend_ptr);
        // 将旧的top chunk放回链表中
        DBG(GET_SIZE(top_chunk) >= MIN_CHUNKSIZE && "mm_malloc: chunk大小不可能小于最小值");
        insertChunk2List(top_chunk);
        top_chunk = extend_ptr;
        // 再次查找
        targetMemPtr = mm_malloc(size);
        //printf("address: %p, chunkSize: %x, reqSize: %x\n", 
          //  targetMemPtr, targetChunksize, size);
        return targetMemPtr;
    }
    ```
    
- mm_free
    
    ```c
    void mm_free(void *bp)
    {   //直接将传入的chunk插入特定索引的链表中即可。插入时自动合并相邻的chunk。
        void* chunk = MEM2HD(bp);
        DBG(GET_ALLOC(chunk) && "mm_free: double Free detected"); // chunk p位为1是已分配，现在set free不是正常吗？
    
        SET_FREE(chunk);
        SET_PREV_FREE(NEXT_CHUNK(chunk));
        DBG(GET_SIZE(chunk) >= MIN_CHUNKSIZE && "mm_free: chunk大小不可能小于最小值");
        insertChunk2List(chunk);
    }
    ```
    
- mm_realloc
    
    ```c
    /*
     * mm_realloc - Implemented simply in terms of mm_malloc and mm_free
    该函数先将传入指针的上下两块可能空闲的chunk合并，然后判断当前chunk的大小是否符合需求。
    如果不符合要求，则分配一块新的内存，复制数据，并最后释放旧的chunk
    如果合并后的chunk大小满足需求，则复制数据并切割多余的空间（如果有多余空间的话）。
     */
    // realloc要额外优化，否则最后两个样例会跑不通
    void *mm_realloc(void *ptr, size_t size)
    {
        DBG(GET_ALLOC(MEM2HD(ptr)) && "mm_realloc: double free / UAF detected");
        //return NULL;
        /*
            void* newp = mm_malloc(size);
            if(newp == NULL)
                return NULL;
            memcpy(newp, ptr, size);
            mm_free(ptr);
            return newp;
        */
    
       // oldptr 和 newptr始终指向chunk的header，而不是mem
        void* oldptr = MEM2HD(ptr);
        void* newptr = oldptr;
        size_t targetChunkSize = request2chunksize(size);
        // 确定要复制的字节数，避免越界读取
        size_t copySize = GET_SIZE(oldptr) - WSIZE;
        if (size < copySize)
            copySize = size;
        
        PRINT_BLOCK(ptr, copySize, "Before realloc:");
    
        // 先合并可能存在的空闲块, 并设置为已分配
        // 这里的合并操作不能使用coalesce，因为这个函数是面向空闲内存的，而oldptr并非空闲
        // newptr = coalesce(oldptr);
    
        int chunksize = GET_SIZE(oldptr);
    
        // 向前合并
        if(GET_PREV_FREE(oldptr))
        {
            void* prevChunk = PREV_CHUNK(oldptr);
            unlink_chunk(prevChunk);
            int prev_chunksize = GET_PREV_SIZE(oldptr);
            chunksize += prev_chunksize;
            newptr = prevChunk;
        }
        // 向后合并
        if(!GET_ALLOC(NEXT_CHUNK(oldptr)) && NEXT_CHUNK(oldptr) != top_chunk)
        {
            chunksize += GET_SIZE(NEXT_CHUNK(oldptr));
            unlink_chunk(NEXT_CHUNK(oldptr));
        }
    
        // 设置大小
        SET_SIZE(newptr, chunksize);
        // 设置被分配
        SET_ALLOC(newptr);
    
        // 如果合并后的大小仍然不够
        if(chunksize < targetChunkSize)
        {
            oldptr = newptr;
            // 分配一块新的
            void* newmem = mm_malloc(size);
            if (newmem == NULL)
                return NULL;
            newptr = MEM2HD(newmem);
            // 复制用户区域
            memcpy(newmem, HD2MEM(oldptr), copySize);
            // 同时释放之前那块
            mm_free(HD2MEM(oldptr));
    
            PRINT_BLOCK(HD2MEM(newptr), copySize, "new alloc:");
        }
        // 如果可以使用
        else
        {
            // 如果不是原来的内存
            if(newptr != oldptr)
            {
                // 复制用户区域
                PRINT_BLOCK(HD2MEM(newptr), copySize, "Before copy newptr :");
                PRINT_BLOCK(HD2MEM(oldptr), copySize, "Before copy oldptr :");
                // 注意这里需要使用memmove，而不是memcpy
                //因为合并后的chunk与原先传入的chunk，其首部可能存在重叠，memmove可以避免这种因为chunk重叠而数据被破坏的错误。
                memmove(HD2MEM(newptr), HD2MEM(oldptr), copySize);
                PRINT_BLOCK(HD2MEM(newptr), copySize, "After copy:");
            }
            if(chunksize >= targetChunkSize + MIN_CHUNKSIZE)
            {
                // 切割，将剩余的存到链表中
                void* remainChunk = splitChunk(newptr, targetChunkSize);
                PRINT_BLOCK(HD2MEM(newptr), copySize, "After Split:");
                insertChunk2List(remainChunk);
            }
    
            PRINT_BLOCK(HD2MEM(newptr), copySize, "After All:");
        }
        return HD2MEM(newptr);
    }
    ```
    

- 调试
    
    ![Untitled](Malloc%20Lab%2084264ff30cef40a0947a0bce7b70e749/Untitled%203.png)
    
    membrk之后无变化
    
    ![Untitled](Malloc%20Lab%2084264ff30cef40a0947a0bce7b70e749/Untitled%204.png)
    

![Untitled](Malloc%20Lab%2084264ff30cef40a0947a0bce7b70e749/Untitled%205.png)