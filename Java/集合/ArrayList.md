# ArrayList

## 主要成员

- elementData[]

- size

## 特点

- 内部使用数组存储元素，get和set的效率高，O(1)

- 通常插入和删除指定位置的元素效率低，涉及到数据的复制转移

- 默认容量为10，直接new ArrayList，这时数组还是空数组，等到第一次add时才会创建容量为10的数组

## grow方法扩容

- 1、将原始容量扩容为1.5倍

- 2、新容量小于最小容量，赋值给新容量

- 3、新容量大于最大容量（Integer.MAX_VALUE - 8），如果最小容量大于最大容量（Integer.MAX_VALUE - 8），新容量就取Integer.MAX_VALUE，否则就取Integer.MAX_VALUE - 8

- 4、调用arrays.copyof创建新数组并进行值的迁移

## add(int index, E element)关键点

将index后面的数据向后移动一位，再赋值给index对应的位置

## remove(int index)关键点

将index后面的数据向前移动一位，然后将数组最后一位数据置为null

## remove(Object o)关键点

先遍历数组找到o对应的index，然后执行和remove(int index)类似的逻辑

## clear()关键点

遍历数组将每个元素置为null

## index(Object o)关键点

- 遍历数组查找，最坏为O(n)

- contains方法内部是调用index方法

## System.arraycopy

重点关注

## arrays.copyof

重点关注