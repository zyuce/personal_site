---
title: "Container_of"
date: 2022-01-15T17:10:18+08:00
draft: true
---



container_of是内核中较为经典的宏定义之一

```c
/** 
 * container_of - cast a member of a structure out to the containing structure 
 * @ptr:    the pointer to the member. 
 * @type:   the type of the container struct this is embedded in. 
 * @member: the name of the member within the struct. 
 * 
 */  
#define container_of(ptr, type, member) ({          \  
    const typeof( ((type *)0)->member ) *__mptr = (ptr); \  
    (type *)( (char *)__mptr - offsetof(type,member) );})  
      
#define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)
```

功能: 根据结构体中的一个成员变量导出包含这个成员变量的struct地址

参数: 

- ptr: 成员变量的地址
- type: 包含成员变量的宿主struct的类型
- member: 成员变量的变量名



以list_head(内核经常使用的双向链表)举例，以下是一个list_head使用场景, my_task_list是链表节点承载的对象

```c
struct  my_task_list {
    int val ;
    struct list_head mylist;
}
```

初始化一个节点:

```c
struct my_task_list my_first_task = 
{ .val = 1,
  .mylist = LIST_HEAD_INIT(my_first_task.mylist)
};
```



通过my_first_task.mylist获取my_first_task:

```c
/**
 * list_entry - get the struct for this entry
 * @ptr:	the &struct list_head pointer.
 * @type:	the type of the struct this is embedded in.
 * @member:	the name of the list_head within the struct.
 */
#define list_entry(ptr, type, member) \
	container_of(ptr, type, member)
	

list_entry(&mylist, struct my_task_list, mylist)
```



```c
   const typeof( ((struct my_task_list *)0)->mylist ) *__mptr = (&first_task.mylist); \  
    (struct my_task_list *)( (char *)__mptr - ((size_t) &((struct my_task_list *)0)->mylist) );})  
```



## 为什么要强制转成char*

因为地址是以字节为单位, 而char的长度就是一个字节