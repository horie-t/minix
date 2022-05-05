# MINIX 3のコードを読みながらコメントを加えていくプロジェクトです。

書籍のバージョン(v3.1.0)をベースにしています。

## 主なディレクトリの説明

| ディレクトリ | 説明 |
| --- | --- |
| kernel | 第一階層(スケジューリング、メッセージ、クロック、システムタスクを担当) |
| drivers | 第二階層(ディスク、コンソール等のドライバ) |
| servers | 第三階層(プロセスマネージャ、ファイルシステム等のサーバ) |
| lib | ライブラリ手続き(open, read等) |
| tools | システム構築用のMakefileやスクリプト |
| boot | 起動、インストール用 |

階層については、書籍を参照。

## カーネルイメージ(ブートイメージ)のビルド

tools/ ディレクトリでビルドする。詳細は [tools/のREADME](./tools/README.md)を参照。

## 起動処理

boot/ ディレクトリで実装されている。詳細は [boot/のREADME](./boot/README.md)を参照。