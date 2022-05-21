# Architecture Lab

![Untitled](Architecture%20Lab%209bc1f65b9f0f4ee6ba852fb9980b06e8/Untitled.png)

- RF: 程序寄存器
- CC: 条件码
- Stat: 程序状态。状态码指明程序是否运行正常或发生某个特殊事件。
- DMEM: 内存
- PC: 程序计数器

- 指令处理框架
    - F**取址：**根据 PC 的值从内存中读取指令字节
        - 指令指示符字节的两个四位部分，为`icode:ifun`
        - 寄存器指示符字节，为 `rA`, `rB`
        - 8字节常数字，为 `valC`
        - 计算下一条指令地址，为 `valP`
    - D**译码：**从寄存器读入最多两个操作数
        - 由 `rA`, `rB` 指明的寄存器，读为 `valA`, `valB`
        - 对于指令`popq`, `pushq`, `call`, `ret`也可能从`%rsp`中读
    - E**执行：**根据`ifun`计算，或计算内存引用的有效地址，或增加或减少栈指针
        - 对上述三者之一进行的操作得到的值为`valE`
        - 如果是计算，则设置条件码
        - 对于条件传送指令，检验条件码和传送条件，并据此更新目标寄存器
        - 对于跳转指令，决定是否选择分支
    - M**访存：**顾名思义
        - 可能是将数据写入内存
        - 若是从内存中读出数据，则读出的值为`valM`
    - W**写回：**最多写两个结果到寄存器
    - **更新 PC：**将 PC 设置成下一条指令的地址

part A:

- **copy.ys :将源块复制到目标块**
    
    ```c
    .pos 0
        irmovq stack, %rsp  # 设置栈指针
        call main           # 执行main函数
        halt                # 程序停止
    
        .align 8            # 八字节对齐
    src:
        .quad 0x1122333
        .quad 0x4455666
        .quad 0x7788999
        .quad 0xaabbccc
        .quad 0xddeefff
    dest:
        .quad 0x123
        .quad 0x456
        .quad 0x789
        .quad 0xabc
        .quad 0xdef
    
    main:
        irmovq src, %rdi    # arg1
        irmovq dest, %rsi   # arg2
        irmovq $5, %rdx     # arg3
        call copy_block     # call
        pushq %rax          # 将返回值存储至栈上
        ret
    
    copy_block:             # src: %rdi   dest: %rsi   len: %rdx
        irmovq $0, %rax     # result = 0
        irmovq $8, %r8      # 每个值八个字节，用来++
        irmovq $1, %r9
        jmp condition
    loop:
        mrmovq (%rdi), %rbx   # val: %rbx     long val = *src
        addq %r8, %rdi        # src++
        rmmovq %rbx, (%rsi)   # *dest = val
        addq %r8, %rsi        # dest++
        xorq %rbx, %rax       # result ^= val
        subq %r9, %rdx        # len--
    condition:
        andq %rdx, %rdx     # 设置标志位，判断len是否大于0
        jg loop
        ret
    
    # 设置初始栈的地址
        .pos 0x200
    stack:
    ```
    
- ****sum.ys：迭代求和链表元素****
    
    ```c
    # 这部分可执行代码设置在内存地址为0的地方。
    # 原因是程序始终会从地址为0的内存处开始执行代码
        .pos 0
        irmovq stack, %rsp  # 设置栈指针
        call main           # 执行main函数
        halt                # 程序停止
    
        .align 8            # 八字节对齐
    node1:
        .quad 0x00001
        .quad node3
    node2:
        .quad 0x00010
        .quad node4
    node3:
        .quad 0x01000
        .quad node5
    node4:
        .quad 0x10000
        .quad 0
    node5:
        .quad 0x00100
        .quad node2
    
    main:
        irmovq node1, %rdi  # arg1
        call sum_list       # call
        pushq %rax          # 将返回值存储至栈上
        ret
    
    sum_list:               # link: %rdi
        irmovq $0, %rax     # 返回值置0
        jmp condition
    loop:
        mrmovq (%rdi), %rbx
        addq %rbx, %rax 
        mrmovq 8(%rdi), %rdi  # node++
    condition:
        andq %rdi, %rdi       # 设置标志位，判断link是否等于NULL
        jne loop
        ret
    
    # 设置初始栈的地址
        .pos 0x200
    stack:
    ```
    
- ****rsum.ys：递归求和链表元素****
    
    ```c
    .pos 0
        irmovq stack, %rsp  # 设置栈指针
        call main           # 执行main函数
        halt                # 程序停止
    
        .align 8            
    node1:
        .quad 0x00001
        .quad node3
    node2:
        .quad 0x00010
        .quad node4
    node3:
        .quad 0x01000
        .quad node5
    node4:
        .quad 0x10000
        .quad 0
    node5:
        .quad 0x00100
        .quad node2
    
    main:
        irmovq node1, %rdi  # arg1
        call rsum_list       # call
        pushq %rax          # 将返回值存储至栈上
        ret
    
    rsum_list:               # link: %rdi
        pushq %r12
        andq %rdi, %rdi     # 判断%rdx等于NULL跳出循环
        je back  
        mrmovq (%rdi), %r12 # 将中间结果保存到 调用者保存 的寄存器
        mrmovq 8(%rdi), %rdi # 设置递归调用的函数参数
        call rsum_list
        addq %r12, %rax     # 递归返回值存中间结果不断回到rax
    back:
        popq %r12
        ret
    
    # 设置初始栈的地址
        .pos 0x200
    stack:
    ```
    

part B:

- iaddq指令指向过程
    
    ```dart
    指令为：iaddq V, rB
    **取指**：
        icode:ifun <- M_1[PC]
        rA:rB <- M_1[PC+1]
        valC <- M_8[PC+2]
        valP <- PC+10
    
    bool instr_valid = // 判断是否是合法指令
    		icode in  { INOP, IHALT, IRRMOVQ, IIRMOVQ, IRMMOVQ, IMRMOVQ,
               IOPQ, IJXX, ICALL, IRET, IPUSHQ, IPOPQ, **IIADDQ** };
    
    bool need_regids =  // 判断指令是否包括寄存器指示符字节, iaddq需要读入一个寄存器rB,因此要额外读取一个字节，故添加到该集合中
        icode in { IRRMOVQ, IOPQ, IPUSHQ, IPOPQ, 
                 IIRMOVQ, IRMMOVQ, IMRMOVQ, **IIADDQ** };
    
    bool need_valC =   // 判断指令是否包括常数字, iaddq需要读入一个常数V,因此添加到该集合中
        icode in { IIRMOVQ, IRMMOVQ, IMRMOVQ, IJXX, ICALL, **IIADDQ** };
    
    **译码：**
        valB <- R[rB]
    
    word srcB = [ // 赋值给valB的寄存器。译码阶段要从rA, rB 指明的寄存器读为 valA, valB，iaddq需要读取右寄存器的值，因此加入到该集合中
        icode in { IOPQ, IRMMOVQ, IMRMOVQ, **IIADDQ**  } : rB;
        icode in { IPUSHQ, IPOPQ, ICALL, IRET } : RRSP;
        1 : RNONE;  # Don't need register
    ];
    
    word dstE = [ // 表明写端口E的目的寄存器，计算出来的值valE将放在那里。最终结果要存放在rB中，这里iaddq指令设置将结果写入右寄存器中
        icode in { IRRMOVQ } && Cnd : rB;
        icode in { IIRMOVQ, IOPQ, **IIADDQ** } : rB;
        icode in { IPUSHQ, IPOPQ, ICALL, IRET } : RRSP;
        1 : RNONE;  # Don't write any register
    ];
    
    **执行：**
        valE <- valB+ valC //iaddq执行阶段进行的运算是valB + valC
        Set CC
    
    ## Select input A to ALU
    word aluA = [    // iaddq设置左操作数为读入的常数项
        icode in { IRRMOVQ, IOPQ } : valA;
        icode in { IIRMOVQ, IRMMOVQ, IMRMOVQ, **IIADDQ** } : valC;
        icode in { ICALL, IPUSHQ } : -8;
        icode in { IRET, IPOPQ } : 8;
        # Other instructions don't need ALU
    ];
    
    ## Select input B to ALU
    word aluB = [  //iaddq设置左操作数为读入的寄存器rB
        icode in { IRMMOVQ, IMRMOVQ, IOPQ, ICALL, 
                  IPUSHQ, IRET, IPOPQ, **IIADDQ** } : valB;
    		// 将立即数存到特定寄存器的操作，就是先运算立即数 + 0，再将结果存入寄存器
        icode in { IRRMOVQ, IIRMOVQ } : 0;
        # Other instructions don't need ALU
    ];
    bool set_cc = icode in { IOPQ, **IIADDQ** }; //判断是否应该更新条件码寄存器
    
    **访存：**
    		// iaddq没有访存阶段
    
    **写回：**
        R[rB] <- valE
    
    **更新PC：**
        PC <- valP //iaddq不涉及转移等操作
    ```
    

part C:

- 代码优化
    - 代码移动：将循环不变量从循环中提出
        
        ```c
        // 优化前
        for(size_t i = 0; i < strlen(str); i++)
          Statements;
        // 优化后
        size_t str_len = strlen(str);
        for(size_t i = 0; i < str_len; i++)
          Statements;
        ```
        
    
    - 循环展开 : 一种程序变换，通过增加每次迭代计算的元素数量，减少循环的迭代次数。
        
        ```c
        // 循环展开前
        for(int i = 0; i < lmits; i++)
          acc0 = acc0 OP data[i];
        
        // 循环展开后
        int limits = length - n;
        int i;
        for(i = 0; i < lmits; i += 2)
        {
          acc0 = acc0 OP data[i];
          acc0 = acc0 OP data[i+1];
        }
        for(; i < length; i ++)
          acc0 = acc0 OP data[i];
        ```
        
        • 多个累计变量：对于一个可结合或可交换的合并运算，我们可以通过将一组合并运算分割成两个或更多的部分，并在最后合并结果来提高性能。
        
        ```c
        int limits = length - n;
        int i;
        for(i = 0; i < lmits; i += 2)
        {
          acc0 = acc0 OP data[i];
          acc1 = acc0 OP data[i+1];
        }
        for(; i < length; i ++)
          acc0 = acc0 OP data[i];
        int total = acc0 + acc1;
        ```
        
- ncopy.ys
    
    `ncopy`函数将一个长度为`len`的整型数组`src`复制到一个不重叠的数组`dst`，并返回`src`
    中正数的个数。
    
    ```dart
    #/* $begin ncopy-ys */
    ##################################################################
    # ncopy.ys - Copy a src block of len words to dst.
    # Return the number of positive words (>0) contained in src.
    #
    # Include your name and ID here.
    #
    # Describe how and why you modified the baseline code.
    #
    ##################################################################
    # Do not modify this portion
    # Function prologue.
    # %rdi = src, %rsi = dst, %rdx = len
    ncopy:
    
    ##################################################################
    # You can modify this portion
    	# Loop header
    	xorq %rax,%rax		# count = 0;
    
        iaddq $-13, %rdx
        andq %rdx,%rdx		
    	jl Done10		
    Loop10:	
        mrmovq (%rdi), %r8	
        mrmovq 8(%rdi), %r9	
        mrmovq 16(%rdi), %r10	
        mrmovq 24(%rdi), %r11	
        mrmovq 32(%rdi), %r12	
        mrmovq 40(%rdi), %r13	
        mrmovq 48(%rdi), %r14	
        mrmovq 56(%rdi), %rcx	
        mrmovq 64(%rdi), %rbx	
        mrmovq 72(%rdi), %rbp	
        
        rmmovq %r8, (%rsi)	
    	andq %r8, %r8		
    	jle Npos10_r8		
    	iaddq $1, %rax		
    Npos10_r8:	
        mrmovq 80(%rdi), %r8	
        rmmovq %r9, 8(%rsi)	
        andq %r9, %r9		
    	jle Npos10_r9		
    	iaddq $1, %rax		
    Npos10_r9:	
        mrmovq 88(%rdi), %r9
        rmmovq %r10, 16(%rsi)	
        andq %r10, %r10		
    	jle Npos10_r10		
    	iaddq $1, %rax		
    Npos10_r10:	
        mrmovq 96(%rdi), %r10
        rmmovq %r11, 24(%rsi)	
        andq %r11, %r11		
    	jle Npos10_r11		
    	iaddq $1, %rax		
    Npos10_r11:
        rmmovq %r12, 32(%rsi)	
        andq %r12, %r12		
    	jle Npos10_r12		
    	iaddq $1, %rax		
    Npos10_r12:
        rmmovq %r13, 40(%rsi)	
    	andq %r13, %r13		
    	jle Npos10_r13		
    	iaddq $1, %rax		
    Npos10_r13:	
        rmmovq %r14, 48(%rsi)	
        andq %r14, %r14		
    	jle Npos10_r14		
    	iaddq $1, %rax		
    Npos10_r14:	
        rmmovq %rcx, 56(%rsi)	
        andq %rcx, %rcx		
    	jle Npos10_rcx		
    	iaddq $1, %rax		
    Npos10_rcx:	
        rmmovq %rbx, 64(%rsi)	
        andq %rbx, %rbx		
    	jle Npos10_rbx		
    	iaddq $1, %rax		
    Npos10_rbx:
        rmmovq %rbp, 72(%rsi)	
        andq %rbp, %rbp		
    	jle Npos10_rbp		
    	iaddq $1, %rax		
    Npos10_rbp:
        rmmovq %r8, 80(%rsi)	
        andq %r8, %r8		
    	jle Npos10_r8_1		
    	iaddq $1, %rax		
    Npos10_r8_1:
        rmmovq %r9, 88(%rsi)	
        andq %r9, %r9		
    	jle Npos10_r9_1		
    	iaddq $1, %rax		
    Npos10_r9_1:
        rmmovq %r10, 96(%rsi)	
        andq %r10, %r10		
    	jle Npos10_r10_1		
    	iaddq $1, %rax		
    Npos10_r10_1:
    
    	iaddq $-13, %rdx		
    	iaddq $104, %rdi		
    	iaddq $104, %rsi		
    
    	andq %rdx,%rdx		
    	jg Loop10			
    Done10:
        iaddq $13, %rdx
    
        iaddq $-5, %rdx
        andq %rdx,%rdx		
    	jl Done5		
    Loop5:	
        mrmovq (%rdi), %r8	
        mrmovq 8(%rdi), %r9	
        mrmovq 16(%rdi), %r10	
        mrmovq 24(%rdi), %r11	
        mrmovq 32(%rdi), %r12	
    
        rmmovq %r8, (%rsi)	
    	andq %r8, %r8		
    	jle Npos5_r8		
    	iaddq $1, %rax		
    Npos5_r8:	
        rmmovq %r9, 8(%rsi)	
        andq %r9, %r9		
    	jle Npos5_r9		
    	iaddq $1, %rax		
    Npos5_r9:	
        rmmovq %r10, 16(%rsi)	
        andq %r10, %r10		
    	jle Npos5_r10		
    	iaddq $1, %rax		
    Npos5_r10:	
        rmmovq %r11, 24(%rsi)	
        andq %r11, %r11		
    	jle Npos5_r11		
    	iaddq $1, %rax		
    Npos5_r11:	
        rmmovq %r12, 32(%rsi)	
        andq %r12, %r12		
    	jle Npos5_r12		
    	iaddq $1, %rax		
    Npos5_r12:	
    
    	iaddq $-5, %rdx		
    	iaddq $40, %rdi		
    	iaddq $40, %rsi		
    
    	andq %rdx,%rdx		
    	jg Loop5			
    Done5:
        iaddq $5, %rdx
    
    //循环赋值
    	xorq %rax,%rax      # count = 0;
    	andq %rdx,%rdx		  # len <= 0?
    	jle Done		        # if so, goto Done:
    Loop:	
      mrmovq (%rdi), %r10	# read val from src...
    	rmmovq %r10, (%rsi)	# ...and store it to dst
    	andq %r10, %r10		  # val <= 0?
    	jle Npos		# if so, goto Npos:
    	iaddq $1, %rax		  # count++
    Npos:	
    	iaddq $-1, %rdx		  # len--
    	iaddq $8, %rdi	  	# src++
    	iaddq $8, %rsi	  	# dst++
    	andq %rdx,%rdx		  # len > 0?
    	jg Loop			# if so, goto Loop:
    ##################################################################
    # Do not modify the following section of code
    # Function epilogue.
    Done:
        ret
    ##################################################################
    # Keep the following label at the end of your function
    End:
    #/* $end ncopy-ys */
    ```
    
- pipe-full.hcl
    
    ```dart
    #/* $begin pipe-all-hcl */
    ####################################################################
    #    HCL Description of Control for Pipelined Y86-64 Processor     #
    #    Copyright (C) Randal E. Bryant, David R. O'Hallaron, 2014     #
    ####################################################################
    
    ## Your task is to implement the iaddq instruction
    ## The file contains a declaration of the icodes
    ## for iaddq (IIADDQ)
    ## Your job is to add the rest of the logic to make it work
    
    ####################################################################
    #    C Include's.  Don't alter these                               #
    ####################################################################
    
    quote '#include <stdio.h>'
    quote '#include "isa.h"'
    quote '#include "pipeline.h"'
    quote '#include "stages.h"'
    quote '#include "sim.h"'
    quote 'int sim_main(int argc, char *argv[]);'
    quote 'int main(int argc, char *argv[]){return sim_main(argc,argv);}'
    
    ####################################################################
    #    Declarations.  Do not change/remove/delete any of these       #
    ####################################################################
    
    ##### Symbolic representation of Y86-64 Instruction Codes #############
    wordsig INOP 	'I_NOP'
    wordsig IHALT	'I_HALT'
    wordsig IRRMOVQ	'I_RRMOVQ'
    wordsig IIRMOVQ	'I_IRMOVQ'
    wordsig IRMMOVQ	'I_RMMOVQ'
    wordsig IMRMOVQ	'I_MRMOVQ'
    wordsig IOPQ	'I_ALU'
    wordsig IJXX	'I_JMP'
    wordsig ICALL	'I_CALL'
    wordsig IRET	'I_RET'
    wordsig IPUSHQ	'I_PUSHQ'
    wordsig IPOPQ	'I_POPQ'
    # Instruction code for iaddq instruction
    wordsig IIADDQ	'I_IADDQ'
    
    ##### Symbolic represenations of Y86-64 function codes            #####
    wordsig FNONE    'F_NONE'        # Default function code
    
    ##### Symbolic representation of Y86-64 Registers referenced      #####
    wordsig RRSP     'REG_RSP'    	     # Stack Pointer
    wordsig RNONE    'REG_NONE'   	     # Special value indicating "no register"
    
    ##### ALU Functions referenced explicitly ##########################
    wordsig ALUADD	'A_ADD'		     # ALU should add its arguments
    
    ##### Possible instruction status values                       #####
    wordsig SBUB	'STAT_BUB'	# Bubble in stage
    wordsig SAOK	'STAT_AOK'	# Normal execution
    wordsig SADR	'STAT_ADR'	# Invalid memory address
    wordsig SINS	'STAT_INS'	# Invalid instruction
    wordsig SHLT	'STAT_HLT'	# Halt instruction encountered
    
    ##### Signals that can be referenced by control logic ##############
    
    ##### Pipeline Register F ##########################################
    
    wordsig F_predPC 'pc_curr->pc'	     # Predicted value of PC
    
    ##### Intermediate Values in Fetch Stage ###########################
    
    wordsig imem_icode  'imem_icode'      # icode field from instruction memory
    wordsig imem_ifun   'imem_ifun'       # ifun  field from instruction memory
    wordsig f_icode	'if_id_next->icode'  # (Possibly modified) instruction code
    wordsig f_ifun	'if_id_next->ifun'   # Fetched instruction function
    wordsig f_valC	'if_id_next->valc'   # Constant data of fetched instruction
    wordsig f_valP	'if_id_next->valp'   # Address of following instruction
    boolsig imem_error 'imem_error'	     # Error signal from instruction memory
    boolsig instr_valid 'instr_valid'    # Is fetched instruction valid?
    
    ##### Pipeline Register D ##########################################
    wordsig D_icode 'if_id_curr->icode'   # Instruction code
    wordsig D_rA 'if_id_curr->ra'	     # rA field from instruction
    wordsig D_rB 'if_id_curr->rb'	     # rB field from instruction
    wordsig D_valP 'if_id_curr->valp'     # Incremented PC
    
    ##### Intermediate Values in Decode Stage  #########################
    
    wordsig d_srcA	 'id_ex_next->srca'  # srcA from decoded instruction
    wordsig d_srcB	 'id_ex_next->srcb'  # srcB from decoded instruction
    wordsig d_rvalA 'd_regvala'	     # valA read from register file
    wordsig d_rvalB 'd_regvalb'	     # valB read from register file
    
    ##### Pipeline Register E ##########################################
    wordsig E_icode 'id_ex_curr->icode'   # Instruction code
    wordsig E_ifun  'id_ex_curr->ifun'    # Instruction function
    wordsig E_valC  'id_ex_curr->valc'    # Constant data
    wordsig E_srcA  'id_ex_curr->srca'    # Source A register ID
    wordsig E_valA  'id_ex_curr->vala'    # Source A value
    wordsig E_srcB  'id_ex_curr->srcb'    # Source B register ID
    wordsig E_valB  'id_ex_curr->valb'    # Source B value
    wordsig E_dstE 'id_ex_curr->deste'    # Destination E register ID
    wordsig E_dstM 'id_ex_curr->destm'    # Destination M register ID
    
    ##### Intermediate Values in Execute Stage #########################
    wordsig e_valE 'ex_mem_next->vale'	# valE generated by ALU
    boolsig e_Cnd 'ex_mem_next->takebranch' # Does condition hold?
    wordsig e_dstE 'ex_mem_next->deste'      # dstE (possibly modified to be RNONE)
    
    ##### Pipeline Register M                  #########################
    wordsig M_stat 'ex_mem_curr->status'     # Instruction status
    wordsig M_icode 'ex_mem_curr->icode'	# Instruction code
    wordsig M_ifun  'ex_mem_curr->ifun'	# Instruction function
    wordsig M_valA  'ex_mem_curr->vala'      # Source A value
    wordsig M_dstE 'ex_mem_curr->deste'	# Destination E register ID
    wordsig M_valE  'ex_mem_curr->vale'      # ALU E value
    wordsig M_dstM 'ex_mem_curr->destm'	# Destination M register ID
    boolsig M_Cnd 'ex_mem_curr->takebranch'	# Condition flag
    boolsig dmem_error 'dmem_error'	        # Error signal from instruction memory
    
    ##### Intermediate Values in Memory Stage ##########################
    wordsig m_valM 'mem_wb_next->valm'	# valM generated by memory
    wordsig m_stat 'mem_wb_next->status'	# stat (possibly modified to be SADR)
    
    ##### Pipeline Register W ##########################################
    wordsig W_stat 'mem_wb_curr->status'     # Instruction status
    wordsig W_icode 'mem_wb_curr->icode'	# Instruction code
    wordsig W_dstE 'mem_wb_curr->deste'	# Destination E register ID
    wordsig W_valE  'mem_wb_curr->vale'      # ALU E value
    wordsig W_dstM 'mem_wb_curr->destm'	# Destination M register ID
    wordsig W_valM  'mem_wb_curr->valm'	# Memory M value
    
    ####################################################################
    #    Control Signal Definitions.                                   #
    ####################################################################
    
    ################ Fetch Stage     ###################################
    
    ## What address should instruction be fetched at
    word f_pc = [
    	# Mispredicted branch.  Fetch at incremented PC
    	M_icode == IJXX && !M_Cnd : M_valA;
    	# Completion of RET instruction
    	W_icode == IRET : W_valM;
    	# Default: Use predicted value of PC
    	1 : F_predPC;
    ];
    
    ## Determine icode of fetched instruction
    word f_icode = [
    	imem_error : INOP;
    	1: imem_icode;
    ];
    
    # Determine ifun
    word f_ifun = [
    	imem_error : FNONE;
    	1: imem_ifun;
    ];
    
    # Is instruction valid?
    bool instr_valid = f_icode in 
    	{ INOP, IHALT, IRRMOVQ, IIRMOVQ, IRMMOVQ, IMRMOVQ,
    	  IOPQ, IJXX, ICALL, IRET, IPUSHQ, IPOPQ, IIADDQ };
    
    # Determine status code for fetched instruction
    word f_stat = [
    	imem_error: SADR;
    	!instr_valid : SINS;
    	f_icode == IHALT : SHLT;
    	1 : SAOK;
    ];
    
    # Does fetched instruction require a regid byte?
    bool need_regids =
    	f_icode in { IRRMOVQ, IOPQ, IPUSHQ, IPOPQ, 
    		     IIRMOVQ, IRMMOVQ, IMRMOVQ, IIADDQ };
    
    # Does fetched instruction require a constant word?
    bool need_valC =
    	f_icode in { IIRMOVQ, IRMMOVQ, IMRMOVQ, IJXX, ICALL, IIADDQ };
    
    # Predict next value of PC
    word f_predPC = [
    	f_icode in { IJXX, ICALL } : f_valC;
    	1 : f_valP;
    ];
    
    ################ Decode Stage ######################################
    
    ## What register should be used as the A source?
    word d_srcA = [
    	D_icode in { IRRMOVQ, IRMMOVQ, IOPQ, IPUSHQ  } : D_rA;
    	D_icode in { IPOPQ, IRET } : RRSP;
    	1 : RNONE; # Don't need register
    ];
    
    ## What register should be used as the B source?
    word d_srcB = [
    	D_icode in { IOPQ, IRMMOVQ, IMRMOVQ, IIADDQ  } : D_rB;
    	D_icode in { IPUSHQ, IPOPQ, ICALL, IRET } : RRSP;
    	1 : RNONE;  # Don't need register
    ];
    
    ## What register should be used as the E destination?
    word d_dstE = [
    	D_icode in { IRRMOVQ, IIRMOVQ, IOPQ, IIADDQ} : D_rB;
    	D_icode in { IPUSHQ, IPOPQ, ICALL, IRET } : RRSP;
    	1 : RNONE;  # Don't write any register
    ];
    
    ## What register should be used as the M destination?
    word d_dstM = [
    	D_icode in { IMRMOVQ, IPOPQ } : D_rA;
    	1 : RNONE;  # Don't write any register
    ];
    
    ## What should be the A value?
    ## Forward into decode stage for valA
    word d_valA = [
    	D_icode in { ICALL, IJXX } : D_valP; # Use incremented PC
    	d_srcA == e_dstE : e_valE;    # Forward valE from execute
    	d_srcA == M_dstM : m_valM;    # Forward valM from memory
    	d_srcA == M_dstE : M_valE;    # Forward valE from memory
    	d_srcA == W_dstM : W_valM;    # Forward valM from write back
    	d_srcA == W_dstE : W_valE;    # Forward valE from write back
    	1 : d_rvalA;  # Use value read from register file
    ];
    
    word d_valB = [
    	d_srcB == e_dstE : e_valE;    # Forward valE from execute
    	d_srcB == M_dstM : m_valM;    # Forward valM from memory
    	d_srcB == M_dstE : M_valE;    # Forward valE from memory
    	d_srcB == W_dstM : W_valM;    # Forward valM from write back
    	d_srcB == W_dstE : W_valE;    # Forward valE from write back
    	1 : d_rvalB;  # Use value read from register file
    ];
    
    ################ Execute Stage #####################################
    
    ## Select input A to ALU
    word aluA = [
    	E_icode in { IRRMOVQ, IOPQ } : E_valA;
    	E_icode in { IIRMOVQ, IRMMOVQ, IMRMOVQ, IIADDQ } : E_valC;
    	E_icode in { ICALL, IPUSHQ } : -8;
    	E_icode in { IRET, IPOPQ } : 8;
    	# Other instructions don't need ALU
    ];
    
    ## Select input B to ALU
    word aluB = [
    	E_icode in { IRMMOVQ, IMRMOVQ, IOPQ, ICALL, 
    		     IPUSHQ, IRET, IPOPQ, IIADDQ } : E_valB;
    	E_icode in { IRRMOVQ, IIRMOVQ } : 0;
    	# Other instructions don't need ALU
    ];
    
    ## Set the ALU function
    word alufun = [
    	E_icode == IOPQ : E_ifun;
    	1 : ALUADD;
    ];
    
    ## Should the condition codes be updated?
    bool set_cc = E_icode in { IOPQ, IIADDQ } &&
    	# State changes only during normal operation
    	!m_stat in { SADR, SINS, SHLT } && !W_stat in { SADR, SINS, SHLT };
    
    ## Generate valA in execute stage
    word e_valA = E_valA;    # Pass valA through stage
    
    ## Set dstE to RNONE in event of not-taken conditional move
    word e_dstE = [
    	E_icode == IRRMOVQ && !e_Cnd : RNONE;
    	1 : E_dstE;
    ];
    
    ################ Memory Stage ######################################
    
    ## Select memory address
    word mem_addr = [
    	M_icode in { IRMMOVQ, IPUSHQ, ICALL, IMRMOVQ } : M_valE;
    	M_icode in { IPOPQ, IRET } : M_valA;
    	# Other instructions don't need address
    ];
    
    ## Set read control signal
    bool mem_read = M_icode in { IMRMOVQ, IPOPQ, IRET };
    
    ## Set write control signal
    bool mem_write = M_icode in { IRMMOVQ, IPUSHQ, ICALL };
    
    #/* $begin pipe-m_stat-hcl */
    ## Update the status
    word m_stat = [
    	dmem_error : SADR;
    	1 : M_stat;
    ];
    #/* $end pipe-m_stat-hcl */
    
    ## Set E port register ID
    word w_dstE = W_dstE;
    
    ## Set E port value
    word w_valE = W_valE;
    
    ## Set M port register ID
    word w_dstM = W_dstM;
    
    ## Set M port value
    word w_valM = W_valM;
    
    ## Update processor status
    word Stat = [
    	W_stat == SBUB : SAOK;
    	1 : W_stat;
    ];
    
    ################ Pipeline Register Control #########################
    
    # Should I stall or inject a bubble into Pipeline Register F?
    # At most one of these can be true.
    bool F_bubble = 0;
    bool F_stall =
    	# Conditions for a load/use hazard
    	E_icode in { IMRMOVQ, IPOPQ } &&
    	 E_dstM in { d_srcA, d_srcB } ||
    	# Stalling at fetch while ret passes through pipeline
    	IRET in { D_icode, E_icode, M_icode };
    
    # Should I stall or inject a bubble into Pipeline Register D?
    # At most one of these can be true.
    bool D_stall = 
    	# Conditions for a load/use hazard
    	E_icode in { IMRMOVQ, IPOPQ } &&
    	 E_dstM in { d_srcA, d_srcB };
    
    bool D_bubble =
    	# Mispredicted branch
    	(E_icode == IJXX && !e_Cnd) ||
    	# Stalling at fetch while ret passes through pipeline
    	# but not condition for a load/use hazard
    	!(E_icode in { IMRMOVQ, IPOPQ } && E_dstM in { d_srcA, d_srcB }) &&
    	  IRET in { D_icode, E_icode, M_icode };
    
    # Should I stall or inject a bubble into Pipeline Register E?
    # At most one of these can be true.
    bool E_stall = 0;
    bool E_bubble =
    	# Mispredicted branch
    	(E_icode == IJXX && !e_Cnd) ||
    	# Conditions for a load/use hazard
    	E_icode in { IMRMOVQ, IPOPQ } &&
    	 E_dstM in { d_srcA, d_srcB};
    
    # Should I stall or inject a bubble into Pipeline Register M?
    # At most one of these can be true.
    bool M_stall = 0;
    # Start injecting bubbles as soon as exception passes through memory stage
    bool M_bubble = m_stat in { SADR, SINS, SHLT } || W_stat in { SADR, SINS, SHLT };
    
    # Should I stall or inject a bubble into Pipeline Register W?
    bool W_stall = W_stat in { SADR, SINS, SHLT };
    bool W_bubble = 0;
    #/* $end pipe-all-hcl */
    ```