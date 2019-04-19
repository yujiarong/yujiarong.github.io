---
title: yield 单向链表操作
date: 2018-03-08 
categories: 算法
tag: 链表
---

### PHP 操作单向链表

``` bash
class SingleLinkedListNode
{
    /**
     * 节点中的数据域
     *
     * @var null
     */
    public $data;

    /**
     * 节点中的指针域，指向下一个节点
     *
     * @var SingleLinkedListNode
     */
    public $next;

    /**
     * SingleLinkedListNode constructor.
     *
     * @param null $data
     */
    public function __construct($data = null)
    {
        $this->data = $data;
        $this->next = null;
    }
}
```

``` bash
class SingleLinkedList
{
    /**
     * 单链表头结点（哨兵节点）
     *
     * @var SingleLinkedListNode
     */
    public $head;

    /**
     * 单链表长度
     *
     * @var
     */
    private $length;

    /**
     * 初始化单链表
     *
     * SingleLinkedList constructor.
     *
     * @param null $head
     */
    public function __construct($head = null)
    {
        if (null == $head) {
            $this->head = new SingleLinkedListNode();
        } else {
            $this->head = $head;
        }

        $this->length = 0;
    }

    /**
     * 获取链表长度
     *
     * @return int
     */
    public function getLength()
    {
        return $this->length;
    }

    /**
     * 插入数据 采用头插法 插入新数据
     *
     * @param $data
     *
     * @return SingleLinkedListNode|bool
     */
    public function insert($data)
    {
        return $this->insertDataAfter($this->head, $data);
    }

    /**
     * 删除节点
     *
     * @param SingleLinkedListNode $node
     *
     * @return bool
     */
    public function delete(SingleLinkedListNode $node)
    {
        if (null == $node) {
            return false;
        }

        // 获取待删除节点的前置节点
        $preNode = $this->getPreNode($node);

        // 修改指针指向
        $preNode->next = $node->next;
        unset($node);

        $this->length--;
        return true;
    }

    /**
     * 通过索引获取节点
     *
     * @param int $index
     *
     * @return SingleLinkedListNode|null
     */
    public function getNodeByIndex($index)
    {
        if ($index >= $this->length) {
            return null;
        }

        $cur = $this->head->next;
        for ($i = 0; $i < $index; ++$i) {
            $cur = $cur->next;
        }

        return $cur;
    }

    /**
     * 获取某个节点的前置节点
     *
     * @param SingleLinkedListNode $node
     *
     * @return SingleLinkedListNode|bool
     */
    public function getPreNode(SingleLinkedListNode $node)
    {
        if (null == $node) {
            return false;
        }

        $curNode = $this->head;
        $preNode = $this->head;
        // 遍历找到前置节点 要用全等判断是否是同一个对象
        // http://php.net/manual/zh/language.oop5.object-comparison.php
        while ($curNode !== $node && $curNode != null) {
            $preNode = $curNode;
            $curNode = $curNode->next;
        }

        return $preNode;
    }

    /**
     * 输出单链表 当data的数据为可输出类型
     *
     * @return bool
     */
    public function printList()
    {
        if (null == $this->head->next) {
            return false;
        }

        $curNode = $this->head;
        // 防止链表带环，控制遍历次数
        $listLength = $this->getLength();
        while ($curNode->next != null && $listLength--) {
            echo $curNode->next->data . ' -> ';

            $curNode = $curNode->next;
        }
        echo 'NULL' . PHP_EOL;

        return true;
    }


    /**
     * 单链表反转
     *
     */
    public function reverse(){
        if( $this->head->next == null){
            return false;
        }
        $curNode = $this->head->next;
        $pre    = null;
        while($curNode  != null){
            $tmp = $curNode->next;
            $curNode->next  = $pre;
            $pre  = $curNode;
            $curNode= $tmp;

        }
        $this->head->next = $pre;
    }


    /**
     * 输出单链表 当data的数据为可输出类型
     *
     * @return bool
     */
    public function printListSimple()
    {
        if (null == $this->head->next) {
            return false;
        }

        $curNode = $this->head;
        while ($curNode->next != null) {
            echo $curNode->next->data . ' -> ';

            $curNode = $curNode->next;
        }
        echo 'NULL' . PHP_EOL;

        return true;
    }

    /**
     * 在某个节点后插入新的节点 (直接插入数据)
     *
     * @param SingleLinkedListNode $originNode
     * @param                      $data
     *
     * @return SingleLinkedListNode|bool
     */
    public function insertDataAfter(SingleLinkedListNode $originNode, $data)
    {
        // 如果originNode为空，插入失败
        if (null == $originNode) {
            return false;
        }

        // 新建单链表节点
        $newNode = new SingleLinkedListNode();
        // 新节点的数据
        $newNode->data = $data;

        // 新节点的下一个节点为源节点的下一个节点
        $newNode->next = $originNode->next;
        // 在originNode后插入newNode
        $originNode->next = $newNode;

        // 链表长度++
        $this->length++;

        return $newNode;
    }

    /**
     * 在某个节点前插入新的节点（很少使用）
     *
     * @param SingleLinkedListNode $originNode
     * @param                      $data
     *
     * @return SingleLinkedListNode|bool
     */
    public function insertDataBefore(SingleLinkedListNode $originNode, $data)
    {
        // 如果originNode为空，插入失败
        if (null == $originNode) {
            return false;
        }

        // 先找到originNode的前置节点，然后通过insertDataAfter插入
        $preNode = $this->getPreNode($originNode);

        return $this->insertDataAfter($preNode, $data);
    }

    /**
     * 在某个节点后插入新的节点
     *
     * @param SingleLinkedListNode $originNode
     * @param SingleLinkedListNode $node
     *
     * @return SingleLinkedListNode|bool
     */
    public function insertNodeAfter(SingleLinkedListNode $originNode, SingleLinkedListNode $node)
    {
        // 如果originNode为空，插入失败
        if (null == $originNode) {
            return false;
        }

        $node->next = $originNode->next;
        $originNode->next = $node;

        $this->length++;

        return $node;
    }

    /**
     * 构造一个有环的链表
     */
    public function buildHasCircleList()
    {
        $data = [1, 2, 3, 4, 5, 6, 7, 8];

        $node0 = new SingleLinkedListNode($data[0]);
        $node1 = new SingleLinkedListNode($data[1]);
        $node2 = new SingleLinkedListNode($data[2]);
        $node3 = new SingleLinkedListNode($data[3]);
        $node4 = new SingleLinkedListNode($data[4]);
        $node5 = new SingleLinkedListNode($data[5]);
        $node6 = new SingleLinkedListNode($data[6]);
        $node7 = new SingleLinkedListNode($data[7]);

        $this->insertNodeAfter($this->head, $node0);
        $this->insertNodeAfter($node0, $node1);
        $this->insertNodeAfter($node1, $node2);
        $this->insertNodeAfter($node2, $node3);
        $this->insertNodeAfter($node3, $node4);
        $this->insertNodeAfter($node4, $node5);
        $this->insertNodeAfter($node5, $node6);
        $this->insertNodeAfter($node6, $node7);

        $node7->next = $node4;
    }

    /**
     * 判断链表是否有环 使用快慢指针
     */

    public function detectCycle() {
        if($this->head == null ||$this->head->next->next== null ||$this->head->next == null ){
            return false;
        }
        $node1 = $this->head->next->next;
        $node2 = $this->head->next;
        while ($node1 !== null && $node2 !== null) {
            if ($node1 === $node2) {
                return true;
            }
            if( !isset($node1->next->next) || !isset($node2->next)  ){
                return false;
            }
            $node1 = $node1->next->next;
            $node2 = $node2->next;
        }
        return false;

    }
}
```
### 合并两个有序的单向链表
逻辑和2个有序数据归并差不多，只是链表操作需要注意。模拟2个有序链表的时候 如果用 list2 = $list  会出现循环引用 很刺激 
``` bash
//模拟2个 有序链表
$list = new SingleLinkedList();
$list->insert(10);
$list->insert(8);
$list->insert(6);
$list->insert(5);
$list->insert(2);
$list->insert(1);
$list2 = new SingleLinkedList();
$list2->insert(10);
$list2->insert(8);
$list2->insert(6);
$list2->insert(5);
$list2->insert(2);
$list2->insert(1);
$list->printList();
$list2->printList();

$cur1 = $list->head->next;
$cur2 = $list2->head->next;
$newList = new SingleLinkedList();
$node  = $newList->head;
while($cur1 != null && $cur2 != null  ){
    if($cur1->data  > $cur2->data ){
        $node->next = $cur2 ;
        $node       = $cur2;
        $cur2 = $node->next;
    }else{
        $node->next = $cur1;
        $node       = $cur1;
        $cur1  = $cur1->next;
    }


}
if($cur2!=null){
    $node->next=$cur2;

}
if($cur1!=null){
    $node->next=$cur1;
}
print_r($newList);
```

### 合并2个有序数组
``` bash
<?php
#有序数组归并
$a = [5,8,23,78];
$b = [1,3,4,5,32,54,100];

function mergeSort($a,$b){
	$len_a  = count($a);
	$len_b  = count($b);
	$temp   = [];
	$i = $j = 0;
	while ( $i < $len_a  &&  $j < $len_b ) {
		if($a[$i] > $b[$j]){
			$temp[] = $b[$j];
			$j++;
		}else{
			$temp[] = $a[$i];
			$i++;
		}
	}
	if( $i = $len_a -1){
		while ($j <$len_b) {
			$temp[] = $b[$j];
			$j++;
		}
	}else{
		while ($i <$len_a) {
			$temp[] = $a[$i];
			$i++;
		}		
	}

	return $temp;
}

$data = mergeSort($a,$b);
print_r($data);
?>
```