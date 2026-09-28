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
| `php_version` | `8.4` | PHP 版本 |
| `extensions` | 完整扩展列表 | 编译扩展（逗号分隔） |
| `dl_parallel` | `10` | 下载并行数 |
| `dl_retry` | `5` | 下载重试次数 |
| `dl_ignore_cache` | `php-src` | 忽略下载缓存 |

## 产物

构建完成后会在 Release 中生成：

- `static-php-cli-{php_version}_{timestamp}.zip` — 包含 `buildroot/bin/` 目录内容
- `release.txt` — 构建参数与产物校验信息

## 扩展列表

默认扩展：

```
amqp,apcu,bcmath,calendar,ctype,curl,dba,dom,exif,fileinfo,filter,ftp,gd,gmp,grpc,http,iconv,imagick,intl,mbregex,mbstring,mongodb,msgpack,mysqli,mysqlnd,opcache,openssl,pcntl,pdo,pdo_mysql,pdo_pgsql,pdo_sqlite,pgsql,phar,posix,readline,redis,session,simplexml,sockets,sodium,sqlite3,swoole,sysvmsg,sysvsem,tokenizer,xlswriter,xml,xmlreader,xmlwriter,xsl,zip,zlib
```
