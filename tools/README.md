# ブートイメージの作成

## ビルド方法

```bash
make image
```

でブートイメージが作成される。

処理内容は、まず

```makefile
PROGRAMS=	../kernel/kernel \
	../servers/pm/pm \
	../servers/fs/fs \
	../servers/rs/rs \
	../drivers/tty/tty \
	../drivers/memory/memory \
	../drivers/log/log \
	AT:../drivers/at_wini/at_wini \
	BIOS:../drivers/bios_wini/bios_wini \
	FLOPPY:../drivers/floppy/floppy \
	../servers/init/init \
#	bootdev.img
```

のバイナリファイル(a.out形式)をコンパイルして、これらのプログラム群を `installboot` で一つのブートイメージファイルに結合する。

installbootの使用方法は以下の通り。

```
Usage: installboot -i(mage) image kernel mm fs ... init
```

## ブートイメージの構成

以下のような構成で各項目がセクタサイズ(512バイト)にアライメントされて保存される。

| 内容 |
| --- |
| kernelのa.outヘッダ部 |
| kernelのtext部 |
| kernelのdata部 |
| pmのa.outヘッダ部 |
| pmのtext部 |
| pmのdata部 |
| ... |
| initのa.outヘッダ部 |
| initのtext部 |
| initのdata部 |

a.outの形式によってはtext部とdata部は一緒になる場合もある。