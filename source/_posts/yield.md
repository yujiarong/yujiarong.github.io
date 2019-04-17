---
title: yield 生成器 读取超大文件
date: 2019-02-01 
categories: PHP
tag: yield
---
生成器函数的核心是yield关键字。它最简单的调用形式看起来像一个return申明，不同之处在于普通return会返回值并终止函数的执行，而yield会返回一个值给循环调用此生成器的代码并且只是暂停执行生成器函数。

### 读取超大文件

``` bash
function readTxt($file)
{
    # code...
    $handle = fopen($file, 'rb');

    while (feof($handle)===false) {
        # code...
        yield fgets($handle);
    }

    fclose($handle);
}

```