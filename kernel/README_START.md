# カーネルの初期化と開始について

ブートモニタからは、mpx386.sのMINIX:にジャンプしてくる。

```Assembly
MINIX:				! this is the entry point for the MINIX kernel
	jmp	over_flags	! skip over the next few bytes
	.data2	CLICK_SHIFT	! for the monitor: memory granularity
flags:
	.data2	0x01FD		! boot monitor flags:
				!	call in 386 mode, make bss, make stack,
				!	load high, don't patch, will return,
				!	uses generic INT, memory vector,
				!	new boot code return
	nop			! extra byte to sync up disassembler
over_flags:
```

ひとまず、フラグデータの部分を飛び越える。このflagsはブートモニタの`get_clickshift()`で参照されるのでカーネルでは不要。この場所にあるのは単純にアドレス指定ができるから。

```Assembly
	movzx	esp, sp		! monitor stack is a 16 bit stack
```

スタックポインタを16ビットから32ビットに変換する。(zxをつけてゼロ拡張(Zero eXtend)している)

```Assembly
! Copy the monitor global descriptor table to the address space of kernel and
! switch over to it.  Prot_init() can then update it with immediate effect.

	sgdt	(_gdt+GDT_SELECTOR)		! get the monitor gdtr
	mov	esi, (_gdt+GDT_SELECTOR+2)	! absolute address of GDT
	mov	ebx, _gdt			! address of kernel GDT
	mov	ecx, 8*8			! copying eight descriptors
copygdt:
 eseg	movb	al, (esi)
	movb	(ebx), al
	inc	esi
	inc	ebx
	loop	copygdt
```

モニタのメモリ領域にあるGDTを、カーネル側の領域にコピー。

```Assembly
	mov	eax, (_gdt+DS_SELECTOR+2)	! base of kernel data
	and	eax, 0x00FFFFFF			! only 24 bits
	add	eax, _gdt			! eax = vir2phys(gdt)
	mov	(_gdt+GDT_SELECTOR+2), eax	! set base of GDT
	lgdt	(_gdt+GDT_SELECTOR)		! switch over to kernel GDT
```

コピーしたカーネル側のGDTに切り替え。


```Assembly
! Locate boot parameters, set up kernel segment registers and stack.
	mov	ebx, 8(ebp)	! boot parameters offset
	mov	edx, 12(ebp)	! boot parameters length
```

引数をレジスタに一旦コピー、`cstart`を呼ぶ直前にスタックに積み直す。

```Assembly
	mov	eax, 16(ebp)	! address of a.out headers
	mov	(_aout), eax
```

a.outのヘッダ情報のアドレスをカーネル側の領域にコピー。

```Assembly
	mov	ax, ds		! kernel data
	mov	es, ax
	mov	fs, ax
	mov	gs, ax
	mov	ss, ax
```

セグメント・セレクタを変更。

```Assembly
	mov	esp, k_stktop	! set sp to point to the top of kernel stack
```

スタック領域をカーネル用のメモリ領域に変更。

```Assembly
! Call C startup code to set up a proper environment to run main().
	push	edx
	push	ebx
	push	SS_SELECTOR
	push	DS_SELECTOR
	push	CS_SELECTOR
	call	_cstart		! cstart(cs, ds, mds, parmoff, parmlen)
```

cstartの引数をスタックに積んでから、cstartを呼び出す。

# start.c

## void cstart(cs, ds, mds, parmoff, parmsize)

カーネル情報のグローバル変数(`struct kinfo kinfo`, `struct machine machine`)を設定するのと、`prot_init()`(後で解説)を呼び出して、CPUの保護機構、割込みテーブルの初期化をする。

完了したら、アセンブラのコードに戻る。

# minix386.s

```Assembly
	call	_cstart		! cstart(cs, ds, mds, parmoff, parmlen)
	add	esp, 5*4
```

cstartから戻ったら、呼び出し前にスタックに積んだ引数を無視するようにスタックポインタを調整する。

```Assembly
! Reload gdtr, idtr and the segment registers to global descriptor table set
! up by prot_init().

	lgdt	(_gdt+GDT_SELECTOR)
	lidt	(_gdt+IDT_SELECTOR)

	jmpf	CS_SELECTOR:csinit
csinit:
```

`prot_init()`で設定したグローバルディスクリプタテーブル、割込みディスクリプタテーブルを読み込んで有効化する。有効化後にジャンプして強制的に使用させる。

```Assembly
    o16	mov	ax, DS_SELECTOR
	mov	ds, ax
	mov	es, ax
	mov	fs, ax
	mov	gs, ax
	mov	ss, ax
```
セグメント・セレクタを設定。

```Assembly
    o16	mov	ax, TSS_SELECTOR	! no other TSS is used
	ltr	ax
```

`ltr`(Load Task Register)でタスクレジスタにロードする。

```Assembly
	push	0			! set flags to known good state
	popf				! esp, clear nested task and int enable
```

`popf`でスタックのトップをEFLAGSの下位16ビットにポップしている。つまり0に設定している。

```Assembly
	jmp	_main			! main()
```

やっとカーネルの`main()`を呼び出すところて到達。

# main.c

```c
PUBLIC void main()
{
// 略
  /* Initialize the interrupt controller. */
  intr_init(1);
```

割込みコントローラを初期化。`intr_init()`は後で読む。
```c
  /* Clear the process table. Anounce each slot as empty and set up mappings 
   * for proc_addr() and proc_nr() macros. Do the same for the table with 
   * privilege structures for the system processes. 
   */
  for (rp = BEG_PROC_ADDR, i = -NR_TASKS; rp < END_PROC_ADDR; ++rp, ++i) {
  	rp->p_rts_flags = SLOT_FREE;		/* initialize free slot */
	rp->p_nr = i;				/* proc number from ptr */
        (pproc_addr + NR_TASKS)[i] = rp;        /* proc ptr from number */
  }
  for (sp = BEG_PRIV_ADDR, i = 0; sp < END_PRIV_ADDR; ++sp, ++i) {
	sp->s_proc_nr = NONE;			/* initialize as free */
	sp->s_id = i;				/* priv structure index */
	ppriv_addr[i] = sp;			/* priv ptr from number */
  }
```

プロセスと特権情報のテーブルを初期化する。最初は空なので、`SLOT_FREE`、`NONE`を設定するだけで、構造体の他のメンバは放置している。


```c
  /* Set up proc table entries for processes in boot image.  The stacks of the
   * kernel tasks are initialized to an array in data space.  The stacks
   * of the servers have been added to the data segment by the monitor, so
   * the stack pointer is set to the end of the data segment.  All the
   * processes are in low memory on the 8086.  On the 386 only the kernel
   * is in low memory, the rest is loaded in extended memory.
   */
  /* ブートイメージ内のプロセスのprocテーブルエントリを設定する。
   * カーネルタスクのスタックは、データ空間内の配列に初期化される。
   * モニターによってサーバーのスタックがデータセグメントに追加さ
   * れているため、スタックポインターはデータセグメントの末尾に
   * 設定される。
   * 8086では、すべてのプロセスのメモリが不足している。
   * 386ではカーネルだけが低位メモリにあり、残りは拡張メモリに
   * ロードされる。
   */
  /* Task stacks. */
  ktsb = (reg_t) t_stack;
```

t_stackはtable.cでデータ領域に確保されている
```c
/* Stack space for all the task stacks.  Declared as (char *) to align it. */
#define	TOT_STACK_SPACE	(IDL_S + HRD_S + (2 * TSK_S))
PUBLIC char *t_stack[TOT_STACK_SPACE / sizeof(char *)];
```

定数は以下の通り定義されている。

```c
/* Define stack sizes for the kernel tasks included in the system image. */
#define NO_STACK	0
#define SMALL_STACK	(128 * sizeof(char *))
#define IDL_S	SMALL_STACK	/* 3 intr, 3 temps, 4 db for Intel */
#define	HRD_S	NO_STACK	/* dummy task, uses kernel stack */
#define	TSK_S	SMALL_STACK	/* system and clock task */
```

HRD_Sはカーネルプロセス(ハードウェアプロセスとも呼ぶ)は、独自のスタックを必要としない実装になっているので、0になる。

```c
  for (i=0; i < NR_BOOT_PROCS; ++i) {
	ip = &image[i];				/* process' attributes */
	rp = proc_addr(ip->proc_nr);		/* get process pointer */
	rp->p_max_priority = ip->priority;	/* max scheduling priority */
	rp->p_priority = ip->priority;		/* current priority */
	rp->p_quantum_size = ip->quantum;	/* quantum size in ticks */
	rp->p_ticks_left = ip->quantum;		/* current credit */
	strncpy(rp->p_name, ip->proc_name, P_NAME_LEN); /* set process name */
	(void) get_priv(rp, (ip->flags & SYS_PROC));    /* assign structure */
	priv(rp)->s_flags = ip->flags;			/* process flags */
	priv(rp)->s_trap_mask = ip->trap_mask;		/* allowed traps */
	priv(rp)->s_call_mask = ip->call_mask;		/* kernel call mask */
	priv(rp)->s_ipc_to.chunk[0] = ip->ipc_to;	/* restrict targets */
	if (iskerneln(proc_nr(rp))) {		/* part of the kernel? */ 
		if (ip->stksize > 0) {		/* HARDWARE stack size is 0 */
			rp->p_priv->s_stack_guard = (reg_t *) ktsb;
			*rp->p_priv->s_stack_guard = STACK_GUARD;
		}
		ktsb += ip->stksize;	/* point to high end of stack */
		rp->p_reg.sp = ktsb;	/* this task's initial stack ptr */
		text_base = kinfo.code_base >> CLICK_SHIFT;
					/* processes that are in the kernel */
		hdrindex = 0;		/* all use the first a.out header */
	} else {
		hdrindex = 1 + i-NR_TASKS;	/* servers, drivers, INIT */
	}

	/* The bootstrap loader created an array of the a.out headers at
	 * absolute address 'aout'. Get one element to e_hdr.
	 */
	phys_copy(aout + hdrindex * A_MINHDR, vir2phys(&e_hdr),
						(phys_bytes) A_MINHDR);
	/* Convert addresses to clicks and build process memory map */
	text_base = e_hdr.a_syms >> CLICK_SHIFT;
	text_clicks = (e_hdr.a_text + CLICK_SIZE-1) >> CLICK_SHIFT;
	if (!(e_hdr.a_flags & A_SEP)) text_clicks = 0;	   /* common I&D */
	data_clicks = (e_hdr.a_total + CLICK_SIZE-1) >> CLICK_SHIFT;
	rp->p_memmap[T].mem_phys = text_base;
	rp->p_memmap[T].mem_len  = text_clicks;
	rp->p_memmap[D].mem_phys = text_base + text_clicks;
	rp->p_memmap[D].mem_len  = data_clicks;
	rp->p_memmap[S].mem_phys = text_base + text_clicks + data_clicks;
	rp->p_memmap[S].mem_vir  = data_clicks;	/* empty - stack is in data */

	/* Set initial register values.  The processor status word for tasks 
	 * is different from that of other processes because tasks can
	 * access I/O; this is not allowed to less-privileged processes 
	 */
	rp->p_reg.pc = (reg_t) ip->initial_pc;
	rp->p_reg.psw = (iskernelp(rp)) ? INIT_TASK_PSW : INIT_PSW;

	/* Initialize the server stack pointer. Take it down one word
	 * to give crtso.s something to use as "argc".
	 */
	if (isusern(proc_nr(rp))) {		/* user-space process? */ 
		rp->p_reg.sp = (rp->p_memmap[S].mem_vir +
				rp->p_memmap[S].mem_len) << CLICK_SHIFT;
		rp->p_reg.sp -= sizeof(reg_t);
	}
	
	/* Set ready. The HARDWARE task is never ready. */
	if (rp->p_nr != HARDWARE) {
		rp->p_rts_flags = 0;		/* runnable if no flags */
		lock_enqueue(rp);		/* add to scheduling queues */
	} else {
		rp->p_rts_flags = NO_MAP;	/* prevent from running */
	}

	/* Code and data segments must be allocated in protected mode. */
	alloc_segments(rp);
  }
```

ループが長い…


```c
	ip = &image[i];				/* process' attributes */
// ...
	} else {
		hdrindex = 1 + i-NR_TASKS;	/* servers, drivers, INIT */
	}
```

まずは以下のimage配列からカーネルのプロセス情報を取ってきてプロセスを設定。

```c
  /* システムイメージテーブルには、ブートイメージに含まれるすべてのプログラムが一覧表示される。
   * ここでのエントリの順序は、ブートイメージ内のプログラムの順序と一致しなければならず、すべ
   * てのカーネルタスクが最初に来る必要がある。各エントリには、プロセステーブル用に、
   * プロセス番号、フラグ、クォンタムサイズ (qs) 、スケジューリングキュー、許可トラップ、
   * ipcマスク、および名前が提供される。プログラムカウンターとスタックサイズの初期値は、
   * カーネルタスクに提供される。
   */
PUBLIC struct boot_image image[] = {
/* process nr,   pc, flags, qs,  queue, stack, traps, ipcto, call,  name */ 
 { IDLE,  idle_task, IDL_F,  8, IDLE_Q, IDL_S,     0,     0,     0, "IDLE"  },
 { CLOCK,clock_task, TSK_F, 64, TASK_Q, TSK_S, TSK_T,     0,     0, "CLOCK" },
 { SYSTEM, sys_task, TSK_F, 64, TASK_Q, TSK_S, TSK_T,     0,     0, "SYSTEM"},
 { HARDWARE,      0, TSK_F, 64, TASK_Q, HRD_S,     0,     0,     0, "KERNEL"},
 { PM_PROC_NR,    0, SRV_F, 32,      3, 0,     SRV_T, SRV_M,  PM_C, "pm"    },
 { FS_PROC_NR,    0, SRV_F, 32,      4, 0,     SRV_T, SRV_M,  FS_C, "fs"    },
 { RS_PROC_NR,    0, SRV_F,  4,      3, 0,     SRV_T, SYS_M,  RS_C, "rs"    },
 { TTY_PROC_NR,   0, SRV_F,  4,      1, 0,     SRV_T, SYS_M, DRV_C, "tty"   },
 { MEM_PROC_NR,   0, SRV_F,  4,      2, 0,     SRV_T, DRV_M, MEM_C, "memory"},
 { LOG_PROC_NR,   0, SRV_F,  4,      2, 0,     SRV_T, SYS_M, DRV_C, "log"   },
 { DRVR_PROC_NR,  0, SRV_F,  4,      2, 0,     SRV_T, SYS_M, DRV_C, "driver"},
 { INIT_PROC_NR,  0, USR_F,  8, USER_Q, 0,     USR_T, USR_M,     0, "init"  },
};
```

```c
	/* The bootstrap loader created an array of the a.out headers at
	 * absolute address 'aout'. Get one element to e_hdr.
	 */
     /* ブートストラップローダーは、絶対アドレス 「aout」
      * にa.outヘッダの配列を作成した。e_hdrに1つの要素を取得する。
      */
	phys_copy(aout + hdrindex * A_MINHDR, vir2phys(&e_hdr),
						(phys_bytes) A_MINHDR);
```

ブートストラップ領域で作成した、プロセスのヘッダ情報をカーネル領域にコピーする。

`struct exec e_hdr`の`exec`の定義は以下のようになっている。

```c
struct	exec {			/* a.out header */
  unsigned char	a_magic[2];	/* magic number */
  unsigned char	a_flags;	/* flags, see below */
  unsigned char	a_cpu;		/* cpu id */
  unsigned char	a_hdrlen;	/* length of header */
  unsigned char	a_unused;	/* reserved for future use */
  unsigned short a_version;	/* version stamp (not used at present) */
  long		a_text;		/* size of text segement in bytes */
  long		a_data;		/* size of data segment in bytes */
  long		a_bss;		/* size of bss segment in bytes */
  long		a_entry;	/* entry point */
  long		a_total;	/* total memory allocated */
  long		a_syms;		/* size of symbol table */

  /* SHORT FORM ENDS HERE */
  long		a_trsize;	/* text relocation size */
  long		a_drsize;	/* data relocation size */
  long		a_tbase;	/* text relocation base */
  long		a_dbase;	/* data relocation base */
};
```

```c
	/* Convert addresses to clicks and build process memory map */
	text_base = e_hdr.a_syms >> CLICK_SHIFT;
	text_clicks = (e_hdr.a_text + CLICK_SIZE-1) >> CLICK_SHIFT;
	if (!(e_hdr.a_flags & A_SEP)) text_clicks = 0;	   /* common I&D */
	data_clicks = (e_hdr.a_total + CLICK_SIZE-1) >> CLICK_SHIFT;
	rp->p_memmap[T].mem_phys = text_base;
	rp->p_memmap[T].mem_len  = text_clicks;
	rp->p_memmap[D].mem_phys = text_base + text_clicks;
	rp->p_memmap[D].mem_len  = data_clicks;
	rp->p_memmap[S].mem_phys = text_base + text_clicks + data_clicks;
	rp->p_memmap[S].mem_vir  = data_clicks;	/* empty - stack is in data */
```

ブートストラップでメモリは`CLICK_SHIFT`単位で確保したので、調整してからプロセス情報に登録する。

T, D, Sは以下のように定義。

```c
#define T                  0	/* proc[i].mem_map[T] is for text */
#define D                  1	/* proc[i].mem_map[D] is for data */
#define S                  2	/* proc[i].mem_map[S] is for stack */
```

```c
	/* Set initial register values.  The processor status word for tasks 
	 * is different from that of other processes because tasks can
	 * access I/O; this is not allowed to less-privileged processes 
	 */
	/* 初期レジスタ値を設定する。タスクはI/Oにアクセスできるため、
	 * タスクのプロセッサーステータスワードは他のプロセスのプロセッサー
	 * ステータスワードとは異なる。権限の少ないプロセスには許可されない。
	 */
	rp->p_reg.pc = (reg_t) ip->initial_pc;
	rp->p_reg.psw = (iskernelp(rp)) ? INIT_TASK_PSW : INIT_PSW;
```

```c
	/* Initialize the server stack pointer. Take it down one word
	 * to give crtso.s something to use as "argc".
	 */
	/* サーバースタックポインタを初期化する。crtso.sに"argc"として
	 * 使用するものを与えるために、1ワード分下位アドレスに下げる。
	 */
	if (isusern(proc_nr(rp))) {		/* user-space process? */ 
		rp->p_reg.sp = (rp->p_memmap[S].mem_vir +
				rp->p_memmap[S].mem_len) << CLICK_SHIFT;
		rp->p_reg.sp -= sizeof(reg_t);
	}
```

スタックポインタを設定する。

```c
	/* Set ready. The HARDWARE task is never ready. */
	if (rp->p_nr != HARDWARE) {
		rp->p_rts_flags = 0;		/* runnable if no flags */
		lock_enqueue(rp);		/* add to scheduling queues */
	} else {
		rp->p_rts_flags = NO_MAP;	/* prevent from running */
	}
```
カーネル(ハードウェア)タスク以外はプロセスのキューに入れる。

```c
	/* Code and data segments must be allocated in protected mode. */
	alloc_segments(rp);
```

セグメントの設定をする。(詳細は後で)

```c
  /* We're definitely not shutting down. */
  shutdown_started = 0;

  /* MINIX is now ready. All boot image processes are on the ready queue.
   * Return to the assembly code to start running the current process. 
   */
  bill_ptr = proc_addr(IDLE);		/* it has to point somewhere */
  announce();				/* print MINIX startup banner */
  restart();
```

アセンブラで書かれた`_restart`を実行する。制御がmainに戻らないような実装になっている。

# mpx386.s

## _restart

```Assembly
_restart:

! Restart the current process or the next process if it is set. 

	cmp	(_next_ptr), 0		! see if another process is scheduled
	jz	0f
```

`next_ptr`は`main()`から呼ばれた`lock_enqueue()`の処理の中で設定されているので、ジャンプしない。

```Assembly
	mov 	eax, (_next_ptr)
	mov	(_proc_ptr), eax	! schedule new process 
	mov	(_next_ptr), 0
```

実行するプロセスの情報を設定。

```Assembly
0:	mov	esp, (_proc_ptr)	! will assume P_STACKBASE == 0
	lldt	P_LDT_SEL(esp)		! enable process' segment descriptors 
	lea	eax, P_STACKTOP(esp)	! arrange for next interrupt
	mov	(_tss+TSS3_S_SP0), eax	! to save state in process table
```

コンテキストスイッチを起こすための準備、詳細は後日、調べる。

```Assembly
restart1:
	decb	(_k_reenter)
```
カーネルのリエントリのカウントを減らす。

```Assembly
    o16	pop	gs
    o16	pop	fs
    o16	pop	es
    o16	pop	ds
	popad
	add	esp, 4		! skip return adr
	iretd			! continue process
```

退避していたレジスタをもとに戻し(いつpushした？)、キューに入ったプロセスを開始する。
