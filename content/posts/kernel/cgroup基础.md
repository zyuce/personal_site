---
title: "Cgroup基础"
date: 2022-01-16T17:02:25+08:00
draft: true
---

# 基础

## 相关概念 

1.任务（task）。在 cgroups 中，任务就是系统的一个进程。

2.控制族群（control group）。控制族群就是一组按照某种标准划分的进程。Cgroups 中的资 源控制都是以控制族群为单位实现。一个进程可以加入到某个控制族群，也从一个进程组迁 移到另一个控制族群。一个进程组的进程可以使用 cgroups 以控制族群为单位分配的资源， 同时受到 cgroups 以控制族群为单位设定的限制。

3.层级（hierarchy）。控制族群可以组织成 hierarchical 的形式，既一颗控制族群树。控制族 群树上的子节点控制族群是父节点控制族群的孩子，继承父控制族群的特定的属性。

4.子系统（subsytem）。一个子系统就是一个资源控制器，比如 cpu 子系统就是控制 cpu 时 间分配的一个控制器。子系统必须附加（attach）到一个层级上才能起作用，一个子系统附 加到某个层级以后，这个层级上的所有控制族群都受到这个子系统的控制。

![cgroups层级结构示意图](https://awps-assets.meituan.net/mit-x/blog-images-bundle-2015/3982f44c.png)

## 相互关系 

1.每次在系统中创建新层级时，该系统中的所有任务都是那个层级的默认 cgroup（我们称 之为 root cgroup ，此cgroup在创建层级时自动创建，后面在该层级中创建的cgroup都是此 cgroup的后代）的初始成员。 

2.一个子系统最多只能附加到一个层级。 

3.一个层级可以附加多个子系统。 

4.一个任务可以是多个cgroup的成员，但是这些cgroup必须在不同的层级。 

5.系统中的进程（任务）创建子进程（任务）时，该子任务自动成为其父进程所在 cgroup 的 成员。然后可根据需要将该子任务移动到不同的 cgroup 中，但开始时它总是继承其父任务 的cgroup。



### 个人理解补充:

1.一个cgroup就是一个层级中的某一个节点

2.一个层级可以附加多个子系统，例如cpu、memory可以对应同一个层级



# 细节

## 代码

```c
struct css_set { // 一个task_struct会引用一个css_set
 	atomic_t refcount; // 引用该css_set的进程数 
 	struct hlist_node hlist; // 通过hash表组织的全部css_set，用于查找
 	struct list_head tasks; // 所有引用该css_set的进程的链表
 	struct list_head cg_links; // cg_cgroup_link组成的链表
 	struct cgroup_subsys_state *subsys[CGROUP_SUBSYS_COUNT];
 	struct rcu_head rcu_head; 
};

struct cgroup_subsys_state { // 属于一个特定子系统的cgroup
	struct cgroup *cgroup;
	atomic_t refcnt;
	unsigned long flags;
	struct css_id *id;
};

struct cgroup {
	unsigned long flags;
	atomic_t count;
	struct list_head sibling; // 将同层级cgroup组织成树
	struct list_head children; // 将同层级cgroup组织成树
	struct cgroup *parent; // 将同层级cgroup组织成树
  struct dentry *dentry;
  struct cgroup_subsys_state *subsys[CGROUP_SUBSYS_COUNT];
  struct cgroupfs_root *root; // 所在层级对应的结构体
  struct cgroup *top_cgroup; // 所在层级的根cgroup
  struct list_head css_sets;  // cg_cgroup_link组成的链表
  struct list_head release_list;
  struct list_head pidlists;
  struct mutex pidlist_mutex;
  struct rcu_head rcu_head;
  struct list_head event_list;
  spinlock_t event_list_lock;
};

struct cg_cgroup_link { // cgroup与css_set是多对多的关系，cg_cgroup_link用来表示这种关系
  struct list_head cgrp_link_list; // struct cgroup中css_sets指向的链表
  struct cgroup *cgrp;
  struct list_head cg_link_list; // struct css_set中cg_links指向的链表
  struct css_set *cg;
};
```



# 实用指令

1.mount -t cgroup 查询系统中已经mount的cgroup的文件系统

2.mount -t cgroup -o cpu,cpuset,memory cpu_and_mem /cgroup/cpu_and_mem

层级目录就是根cgroup， cd  /cgroup/cpu_and_mem && mkdir foo 会创建一个子cgroup foo

3.cat tasks 可以看到属于这个cgroup的进程号，根cgroup可以看到系统中所有进程