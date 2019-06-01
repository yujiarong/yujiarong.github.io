---
title:  二分查找与其变体
date: 2018-03-04
categories: 算法
tag: 二分查找
---
二分查找针对的是一个有序的数据集合，时间复杂度为O(logn) 

$data = [8,11,19,23,33,33,33,45,55,67,98];

### 普通的二分查找

``` php
function binarySearch(array $data,$find){
	$left  = 0;
	$right = count($data) - 1;
	while ($left <= $right) {
		$mid = $left + (($right-$left)>>1);
		if($find >$data[$mid]){
			$left  = $mid;
		}else if ($find < $data[$mid]){
			$right = $mid;
		}else{
			return $mid;
		}
	}
	return -1;
}
```

### 找到第一个=find的元素

``` php
function findFirstEqual(array $data,$find) {
    $length = count($data);
    $left = 0;
    $right = $length - 1;
    while($left <= $right) {
        $mid = $left + (($right-$left)>>1);
        if($data[$mid] > $find) {
            $right = $mid - 1;
        }else if($data[$mid] < $find) {
            $left = $mid + 1;
        }else {
            /**
             * 如果是第一个元素，或之前一个元素不等于我们要找的值
             * 我们就找到了第一个=find的element
             */
            if($mid==0 || $data[$mid-1]!=$find) {
                return $mid;
            }else {
                $right = $mid - 1;
            }
        }
    }

    return -1;
}
```

### 找到最后一个=find的元素


``` php
function findLastEqual(array $data,$find) {
    $length = count($data);
    $left = 0;
    $right = $length - 1;
    while($left <= $right) {
        $mid = $left + (($right-$left)>>1);
        if($data[$mid] > $find) {
            $right = $mid - 1;
        }else if($data[$mid] < $find) {
            $left = $mid + 1;
        }else {
            /**
             * 如果mid是最后一个元素的index
             * 或mid后一个元素!=我们要找的值
             * 则找到了最后一个=find的value
             */
            if($mid==$length-1 || $data[$mid+1]!=$find) {
                return $mid;
            }else {
                $left = $mid + 1;
            }
        }
    }

    return -1;
}
```

### 找到第一个大于等于find的元素

``` php
function findFirstGreaterEqual(array $data,$find) {
    $length = count($data);
    $left = 0;
    $right = $length - 1;
    while($left <= $right) {
        $mid = $left + (($right-$left)>>1);
        if($data[$mid] >= $find) {
            if ($mid == 0 || $data[$mid-1] < $find) {
                return $mid;
            }else {
                $right = $mid - 1;
            }
        }else  {
            $left  = $mid + 1;
        }
    }
    return -1;
}
```

### 找到最后一个小于等于find的元素

``` php
function findLastLessEqual(array $data,$find) {
    $length = count($data);
    $left = 0;
    $right = $length - 1;
    while($left <= $right) {
        $mid = $left + (($right-$left)>>1);
        if($data[$mid] <= $find) {
           if($mid==$length-1 || $data[$mid+1]> $find) {
               return $mid;
           }
           $left = $mid + 1;
        }else  {
            $right = $mid - 1;
        }
    }
    return -1;
}

```

