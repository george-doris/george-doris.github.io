---
layout:     post
title:      Sphinx 使用记录
category: blog
description: 工作中需要使用 sphinx 来实现中文全文搜索，最后选择了 sphinx 这个工具，于是记录一下操作步骤。
---


## 安装

安装前需要先去官网下载[源码][sphinxsearch-download].  
目前最新版本是 2.2.5-release, [点击下载][sphinxsearch-download-Source]即可。  

当然，如果你想直接在命令行下载，直接下载我这个版本也行，就是不知道会不会版本太久。

```
tiankonguse:~ $ cd /usr/local/src
tiankonguse:src $ su root -
tiankonguse:src # wget http://sphinxsearch.com/files/sphinx-2.2.5-release.tar.gz
```

然后解压缩，命令就不用说了吧

```
tiankonguse:src # tar zxvf filename.tar.gz
```
 
后来听说 sphinx 有两种安装方式  

1. 单独安装，查询时采用API调用。
2. 使用插件方式把sphinx编译成一个mysql插件并使用特定的sql语句进行检索。

这里我选择第一种方式，毕竟把 sphinx 和 mysql 耦合在一起的话， 将来将成为一个很大的坑。
 
> sphinx 查询出来的是 id, 然后会进行二次查询得到想要的数据。  

下面的命令都是在 root 权限下操作的。

```
tiankonguse:sphinx-2.2.5-release # ./configure –prefix=/usr/local/sphinx

tiankonguse:sphinx-2.2.5-release # make && make install
```

> 可以使用 --prefix 指向sphinx的安装路径
> 可以使用  --with-mysql 指向mysql的安装路径。

安装完毕后查看一下 `/usr/local/sphinx` 下是否有 三个目录 bin etc var，如有，则安装无误！

```
tiankonguse:sphinx-2.2.5-release # cd /usr/local/sphinx/

tiankonguse:sphinx # ls
bin/  etc/  share/  var/
```

## 配置

### mysql 数据源

由于我使用的是 mysql, 所以需要为 sphinx 创建对应的db。

```
# server：127.0.0.1
# database : d_sphinx_testdb
# table: t_sphinx_article

CREATE SCHEMA IF NOT EXISTS `d_sphinx_testdb` DEFAULT CHARACTER SET utf8 COLLATE utf8_general_ci ;

USE `d_sphinx_testdb` ;

CREATE TABLE `d_sphinx_testdb`.`t_sphinx_article` (
  `c_id` INT NOT NULL AUTO_INCREMENT,
  `c_title` VARCHAR(45) NOT NULL DEFAULT '',
  `c_content` VARCHAR(45) NOT NULL DEFAULT '',
  `c_comment_num` VARCHAR(45) NOT NULL DEFAULT 0,
  PRIMARY KEY (`c_id`))
ENGINE = InnoDB
DEFAULT CHARACTER SET = utf8;

```

## sphinx 配置文件

首先需要找到需要配置的文件以及需要配置的内容。

我们需要配置的是 /usr/local/sphinx/sphinx.conf 文件里面的数据库的信息。 

```
tiankonguse:sphinx # cd etc

tiankonguse:etc # 

tiankonguse:etc # ls
example.sql  sphinx-min.conf.dist  sphinx.conf.dist

tiankonguse:etc #  cp sphinx.conf.dist sphinx.conf

tiankonguse:etc # ls
example.sql  sphinx-min.conf.dist  sphinx.conf  sphinx.conf.dist

skyyuan:etc $ vi sphinx.conf
```

可以看到下面的内容设置数据源 source

```
#############################################################################
## data source definition
#############################################################################
 
source d_sphinx_testdb 
{
    # data source type. mandatory, no default value
    # known types are mysql, pgsql, mssql, xmlpipe, xmlpipe2, odbc
    type            = mysql  # 数据库类型
 
    # some straightforward parameters for SQL source types
    #数据库主机地址
    sql_host        = 127.0.0.1 
    
    #数据库用户名
    sql_user        = root      
    
    #数据库密码
    sql_pass        = pwd       
    
    #数据库名称
    sql_db          = d_sphinx_testdb  
    
    # 数据库采用的端口
    sql_port        = 3306  
    
    # pre-query, executed before the main fetch query
    # multi-value, optional, default is empty list of queries
    #执行sql前要设置的字符集 
    sql_query_pre   = SET NAMES UTF8 
    
    # main document fetch query mandatory, integer document ID field MUST be the first selected column
    # 全文检索要显示的内容，在这里尽可能不使用where或group by，将where与groupby的内容交给sphinx，由sphinx进行条件过滤与groupby效率会更高
    # select 出来的字段必须至少包括一个唯一主键(ARTICLESID)以及要全文检索的字段，你计划原本在where中要用到的字段也要select出来，这里不需要使用orderby
    sql_query = SELECT c_id,c_title,c_content,c_comment_num FROM t_sphinx_article

    #####以下是用来过滤或条件查询的属性############

    #sql_attr_ 开头的表示一些属性字段，你原计划要用在where,orderby,groupby中的字段要在这里定义
    
    # unsigned integer attribute declaration
    sql_attr_uint = c_comment_num  # 无符号整数属性
    sql_attr_uint = c_id  # 无符号整数属性

    # boolean attribute declaration
    # sql_attr_bool     = is_deleted
    
    # bigint attribute declaration
    # sql_attr_bigint       = my_bigint_id
    
    # UNIX timestamp attribute declaration
    # sql_attr_timestamp    = posted_ts
    
    # floating point attribute declaration
    # sql_attr_float        = lat_radians
    
    # string attribute declaration
    sql_attr_string       = c_title
    sql_attr_string       = c_content
    
    # JSON attribute declaration
    # sql_attr_json     = properties
    
    # combined field plus attribute declaration (from a single column)
    # stores column as an attribute, but also indexes it as a full-text field
    #
    # sql_field_string  = author
    
}
```

然后设置数据源的索引

```
index d_sphinx_testdb_index
{
    #数据源名
    source = d_sphinx_testdb
    
    # 索引记录存放目录
    path = /usr/local/sphinx/var/data/d_sphinx_testdb_index
    
    # 文档信息存储方式
    docinfo = extern
    
    #缓存数据内存锁定
    mlock = 0
    
    # 形态学
    morphology = none
    
    # 索引的词最小长度
    min_word_len = 1
    
    #数据编码
    charset_type = utf-8
    
    #最小前缀
    min_prefix_len = 0
    
    #最小中缀
    min_infix_len = 1 
}

indexer  
{  
    # 内存限制
    mem_limit  = 32M  
}

searchd  
{  
    # 监听端口
    listen          = 9312  
    
    # 服务进程日志
    log         = /usr/local/sphinx/log/searchd.log  
    
    # 客户端查询日志
    query_log       = /usr/local/sphinx/log/query.log  
    
    # 请求超时
    read_timeout            = 5  
    
    # 同时可执行的最大searchd 进程数
    max_children            = 30  
    
    #进程ID文件
    pid_file        = /usr/local/sphinx/log/searchd.pid 

    # 查询结果的最大返回数    
    max_matches     = 1000  
    
    # 是否支持无缝切换，做增量索引时通常需要
    seamless_rotate         = 1  
}  
```

## 创建索引

进入  bin 目录，执行 

```
./indexer 索引名

Sphinx 2.2.5-id64-release (r4825)
Copyright (c) 2001-2014, Andrew Aksyonoff
Copyright (c) 2008-2014, Sphinx Technologies Inc (http://sphinxsearch.com)

using config file '/usr/local/sphinx/etc/sphinx.conf'...
indexing index 't_cover_sphinx_index'...
collected 1000 docs, 0.4 MB
sorted 0.0 Mhits, 100.0% done
total 1000 docs, 408329 bytes
total 0.041 sec, 9739278 bytes/sec, 23851.54 docs/sec
total 1006 reads, 0.002 sec, 1.0 kb/call avg, 0.0 msec/call avg
total 14 writes, 0.001 sec, 106.9 kb/call avg, 0.1 msec/call avg
```

## 测试

测试前需要安装测试环境，以前 sphinx 的 bin 目录里面有个自带 search 程序，新版本没有了，所以只好使用api方式调用了。

这里我采用 linux + apache + php + sphinx 的方式来完整测试吧。

### apache 源码安装

Apache  的最新版本请参考 [官网下载页面][httpd-apache-download].   
我这里下载的是 [httpd-2.4.10][apache-httpd-2] 最新的 apache 源码版本。

```
wget http://apache.dataguru.cn//httpd/httpd-2.4.10.tar.gz
tar zxvf httpd-2.4.10.tar.gz
cd httpd-2.4.10/
./configure --prefix=/usr/local/apache --with-apr=/usr/local/apr --with-apr-util=/usr/local/apr-util
make && make install
```


### php 源码安装

php 目前的最新版本 [5.6.2][php-source], 建议去[官网下载页][php-home]下载最新版本

```
wget http://cn2.php.net/distributions/php-5.6.2.tar.gz
tar zxvf php-5.6.2.tar.gz
cd php-5.6.2
./configure --prefix=/usr/local/php
make && make install
```

## 错误集

### libmysqlclient.so.18

但是我报下面的错误

```
./indexer: error while loading shared libraries: libmysqlclient.so.18: cannot open shared object file: No such file or directory
```

原因：这主要是因为你安装库后,没有配置相应的环境变量.可以通过连接修正这个问题 

```
sudo ln /usr/local/mysql/lib/libmysqlclient.so.18 /usr/lib/libmysqlclient.so.18 
```

但是还是报错，原来添加一个动态库后需要重新加载动态库。

```
tiankonguse:bin #  ldconfig
```


### Invalid cross-device link

但是我又报错了

```
ln: creating hard link `/usr/lib/libmysqlclient.so.18 ' => `/usr/local/mysql/lib/libmysqlclient.so.18': Invalid cross-device link
```

于是我只好创建软连接了。

```
sudo  ln -s /usr/local/mysql/lib/libmysqlclient.so.18 /usr/lib/libmysqlclient.so.18 
```


### 查看检索是否启动

```
tiankonguse:bin #  ps -ef | grep search
tiankonguse   9601     1  0 Oct28 ?        00:00:00 xs-searchd: master                                             
tiankonguse   9602  9601  0 Oct28 ?        00:00:00 xs-searchd: worker[1]                                          
tiankonguse   9603  9601  0 Oct28 ?        00:00:00 xs-searchd: worker[2]                                          
tiankonguse   9604  9601  0 Oct28 ?        00:00:00 xs-searchd: worker[3]                                          
root     32637 18048  0 21:12 pts/0    00:00:00 grep search
```


### WARNING attribute not found

执行索引的时候，看到这个错误，搜索了一下，原来主键不能加入到属性中去。

```
WARNING: attribute 'c_id' not found - IGNORING
```

参考文档 [数据源配置：mysql数据源][coreseek-products-instal-mysql] 和 [WARNING: zero/NULL document_id, skipping][coreseek-2_948_0] .

### ERROR index No fields in schema


```
ERROR: index 't_cover_sphinx_index': No fields in schema - will not index
```

还是在[这里][coreseek-products-instal-mysql]找到了原因。

> 使用sql_attr设置的字段，只能作为属性，使用SphinxClient::SetFilter()进行过滤；  
> 未被设置的字段，自动作为全文检索的字段，使用SphinxClient::Query("搜索字符串")进行全文搜索


而我把所有字段都设置为 sql_attr 了，于是把需要全文索引的字段去掉。终于跑出一些接过来。

但是还有一些问题。


### WARNING sql_query_info removed from Sphinx

```
WARNING: key 'sql_query_info' was permanently removed from Sphinx configuration. Refer to documentation for details.
```

好吧，我说怎么没有在配置文件中看到 sql_query_info 的说明呢，原来已经删除了，那就注释掉吧。  


### word overrun buffer

还是搜[主键][coreseek-2_1016_0]搜到的原因是我的主键不是一个整数，而 sphinx 要求必须是一个整数。

```
WARNING: source : skipped 300 document(s) with zero/NULL ids
WARNING: word overrun buffer, clipped!!!
WARNING: 601 duplicate document id pairs found
```

###  APR not found

在 config apache 的时候，提示下面的错误。

```
configure: error: APR not found.  Please read the documentation.
```
在[官网的安装页面][apache-install] 可以看到有个 Requirements 类表。

apache 需要依赖于下面的一些东西。

* APR and APR-Util
* Perl-Compatible Regular Expressions Library (PCRE)
* Disk Space
* ANSI-C Compiler and Build System
* Accurate time keeping
* Perl 5 [OPTIONAL]

然后我们可以 [apr][apache-apr] 的官网下载对应的东西即可。

其实就是在 [apr下载页面][apache-apr-download-page] 找到[下载链接][apache-apr-download-source] .

当然我们还要下载 [apr-util][apache-apr-util-download-source] .

#### 安装apr

```
wget http://apache.fayea.com/apache-mirror//apr/apr-1.5.1.tar.gz
tar zxvf  apr-1.5.1.tar.gz
cd apr-1.5.1
./configure --prefix=/usr/local/apr
make && make install

#提示
Libraries have been installed in:
   /usr/local/apr/lib

If you ever happen to want to link against installed libraries
in a given directory, LIBDIR, you must either use libtool, and
specify the full pathname of the library, or use the `-LLIBDIR'
flag during linking and do at least one of the following:
   - add LIBDIR to the `LD_LIBRARY_PATH' environment variable
     during execution
   - add LIBDIR to the `LD_RUN_PATH' environment variable
     during linking
   - use the `-Wl,-rpath -Wl,LIBDIR' linker flag
   - have your system administrator add LIBDIR to `/etc/ld.so.conf'
   
```
#### 安装 apr-util

```
wget http://mirrors.cnnic.cn/apache//apr/apr-util-1.5.4.tar.gz
tar zxvf  apr-util-1.5.4.tar.gz
cd apr-util-1.5.4
./configure --prefix=/usr/local/apr-util

#提示不能找到 APR， 于是指定 APR 的位置
configure: error: APR could not be located. Please use the --with-apr option.


./configure --prefix=/usr/local/apr-util --with-apr=/usr/local/apr/
make && make install

#  运行完后得到下面的提示
Libraries have been installed in:
   /usr/local/apr-util/lib

If you ever happen to want to link against installed libraries
in a given directory, LIBDIR, you must either use libtool, and
specify the full pathname of the library, or use the `-LLIBDIR'
flag during linking and do at least one of the following:
   - add LIBDIR to the `LD_LIBRARY_PATH' environment variable
     during execution
   - add LIBDIR to the `LD_RUN_PATH' environment variable
     during linking
   - use the `-Wl,-rpath -Wl,LIBDIR' linker flag
   - have your system administrator add LIBDIR to `/etc/ld.so.conf'

See any operating system documentation about shared libraries for
more information, such as the ld(1) and ld.so(8) manual pages.
```




[apache-apr-util-download-source]: http://mirrors.cnnic.cn/apache//apr/apr-util-1.5.4.tar.gz
[apache-apr-download-source]: http://apache.fayea.com/apache-mirror//apr/apr-1.5.1.tar.gz
[apache-apr-download-page]: http://apr.apache.org/download.cgi
[apache-apr]: http://apr.apache.org/
[apache-install]: http://httpd.apache.org/docs/2.4/en/install.html
[php-home]: http://cn2.php.net/downloads.php
[php-source]: http://cn2.php.net/distributions/php-5.6.2.tar.gz
[apache-httpd-2]: http://apache.dataguru.cn//httpd/httpd-2.4.10.tar.gz
[httpd-apache-download]: http://httpd.apache.org/download.cgi#apache24
[coreseek-2_1016_0]: http://www.coreseek.cn/forum/2_1016_0.html
[coreseek-products-instal-mysql]: http://www.coreseek.cn/products-install/mysql/
[coreseek-2_948_0]: http://www.coreseek.cn/forum/2_948_0.html
[sphinxsearch-download-Source]: http://sphinxsearch.com/files/sphinx-2.2.5-release.tar.gz 
[sphinxsearch-download]: http://sphinxsearch.com/downloads/release/