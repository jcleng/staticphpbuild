# Static PHP Build

使用 GitHub Actions 构建静态 PHP CLI，产物自动发布到 Release。

## 使用方式

1. 进入仓库的 **Actions** 标签页
2. 选择 **Build Static PHP** 工作流
3. 点击 **Run workflow**，填写参数后执行

## 参数说明

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `spc_url` | `https://dl.static-php.dev/v3/spc-bin/nightly/spc-linux-x86_64` | spc 下载地址 |
| `php_version` | `8.2` | PHP 版本 |
| `extensions` | 完整扩展列表（含 `swoole`） | 编译扩展（逗号分隔） |
| `swoole_custom_git` | `fix/php-snprintf-macro:https://github.com/jcleng/swoole-src.git` | swoole 自定义源码 git（`<branch>:<url>`），使用已修复 snprintf 宏冲突的 fork 源码 |
| `dl_parallel` | `10` | 下载并行数 |
| `dl_retry` | `5` | 下载重试次数 |
| `dl_ignore_cache` | `php-src` | 忽略下载缓存 |

## 产物

构建完成后会在 Release 中生成：

- `static-php-cli-{php_version}_{timestamp}.zip` — 包含 `buildroot/bin/` 目录内容
- `php-{php_version}_{timestamp}` — 独立 php 二进制（从 `buildroot/bin/php` 提取）
- `release.txt` — 构建参数与产物校验信息

## swoole 源码说明

默认 `extensions` 已重新包含 `swoole`。由于 swoole-src v6.2.3 内置的 `nlohmann/json`
与 PHP 的 `#define snprintf ap_php_snprintf` 冲突（见下方 AGENTS.md “Known build failures”），
构建通过 `--dl-custom-git=ext-swoole:<branch>:<url>` 使用已修复的 fork 源码：

- 仓库：`https://github.com/jcleng/swoole-src`
- 分支：`fix/php-snprintf-macro`

如需使用官方未修复源码，可清空 `swoole_custom_git` 输入（但构建会失败，见 AGENTS.md）。

## 扩展列表

默认扩展：

```
amqp,apcu,bcmath,calendar,ctype,curl,dba,dom,exif,fileinfo,filter,ftp,gd,gmp,grpc,http,iconv,imagick,intl,mbregex,mbstring,mongodb,msgpack,mysqli,mysqlnd,opcache,openssl,pcntl,pdo,pdo_mysql,pdo_pgsql,pdo_sqlite,pgsql,phar,posix,readline,redis,session,simplexml,sockets,sodium,sqlite3,sysvmsg,sysvsem,tokenizer,xlswriter,xml,xmlreader,xmlwriter,xsl,zip,zlib,swoole
```
