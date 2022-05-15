# カーネルの初期化と開始について

ブートモニタからは、minix386.sのMINIX:にジャンプしてくる。

```
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

```
	movzx	esp, sp		! monitor stack is a 16 bit stack
```

スタックポインタを16ビットから32ビットに変換する。(zxをつけてゼロ拡張(Zero eXtend)している)

```
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

```
	mov	eax, (_gdt+DS_SELECTOR+2)	! base of kernel data
	and	eax, 0x00FFFFFF			! only 24 bits
	add	eax, _gdt			! eax = vir2phys(gdt)
	mov	(_gdt+GDT_SELECTOR+2), eax	! set base of GDT
	lgdt	(_gdt+GDT_SELECTOR)		! switch over to kernel GDT
```

コピーしたカーネル側のGDTに切り替え。


```
! Locate boot parameters, set up kernel segment registers and stack.
	mov	ebx, 8(ebp)	! boot parameters offset
	mov	edx, 12(ebp)	! boot parameters length
```

引数をレジスタに一旦コピー、`cstart`を呼ぶ直前にスタックに積み直す。

```
	mov	eax, 16(ebp)	! address of a.out headers
	mov	(_aout), eax
```

a.outのヘッダ情報のアドレスをカーネル側の領域にコピー。

```
	mov	ax, ds		! kernel data
	mov	es, ax
	mov	fs, ax
	mov	gs, ax
	mov	ss, ax
```

セグメント・セレクタを変更。

```
	mov	esp, k_stktop	! set sp to point to the top of kernel stack
```

スタック領域をカーネル用のメモリ領域に変更。

```
! Call C startup code to set up a proper environment to run main().
	push	edx
	push	ebx
	push	SS_SELECTOR
	push	DS_SELECTOR
	push	CS_SELECTOR
	call	_cstart		! cstart(cs, ds, mds, parmoff, parmlen)
```

cstartの引数をスタックに積んでから、cstartを呼び出す。
