---
title: InnoDB 物理文件介绍
date: 2020-12-05 20:41:27
categories: 
- InnoDB

---

关于 InnoDB 介绍的文章很多，但是对于 InnoDB 物理文件介绍的文章却很少，本文尝试对 InnoDB 的物理文件结构进行一个简单的说明。

<!-- more -->

## 几个重要概念

### Space

或者叫 table space，表空间，InnoDB 内的表空间有以下几类：系统表空间（ibdata）、临时表空间（ibtmp）、undo表空间（undo001）、用户表空间（ibd）。一个表空间对应一个物理文件，在 InnoDB 最新版本中，默认一个表空间对应一张表。

<img src="/images/space-page-1.png" width="520px" align="left"/>
<img class="nofancybox" src="/images/div.jpg" width="100%" height="0.1px"/>

### Page

页，InnoDB 内文件组织的基本单位，InnoDB 默认单个 Page 的大小是 16KB。单个 Space 由多个 Page 组成，Space 的大小一定是 16KB 的整数倍。Page 的类型比较复杂，包括：FSP_HDR Page、XDES Page、INODE Page、INDEX Page 等。不同类型的 Page 头部和尾部大小是固定的，分别是 38 和 8 字节。

<img src="/images/space-page-2.png"  width="520px" align="left"/>

<img src="/images/space-page-3.png"  width="520px" align="left"/>
<img class="nofancybox" src="/images/div.jpg" width="100%" height="0.1px"/>

### FSP_HDR Page

Space 内的第 1 个 Page，也可以叫做 Page 0。前面提到，Page 是文件组织的基本单位，每 64 个 Page 会组成 1 个 Extent，1 个 Extent 的大小为 1MB；每 256 个 Extent 会有一个单独的 FSP_HDR/XDES 进行管理。

<img src="/images/space-page-4.png"  width="520px" align="left"/>

<img src="/images/space-page-5.png"  width="520px" align="left"/>
<img class="nofancybox" src="/images/div.jpg" width="100%" height="0.1px"/>

注：FREE、FREE_FRAG、FULL_FRAG 列表中的元素为 XDES Entry；FULL_INODES、FREE_INODES 列表中的元素为 INODO Page。

### XDES Page

前面已经提到，每 256 个 Extent 会有一个单独的 FSP_HDR/XDES 进行管理，与 FSP_HDR Page 的区别在于，XDES Page 的 FSP Header 部分为空，其它部分相同。每一个 Extent 由 1 个单独的 XDES Entry 进行管理

<img src="/images/space-page-6.png"  width="520px" align="left"/>
<img class="nofancybox" src="/images/div.jpg" width="100%" height="0.1px"/>

### INODE Page

INODE 这个命名是 InnoDB 里面比较容易让人混淆的概念之一，参考文献中给出了一个说法 file segment INODE，INODE Page 中包含了 85 个 INODE Entry，每个 INODE Entry 描述一个 file segment(FSEG)。

<img src="/images/space-page-7.png"  width="520px" align="left"/>

<img src="/images/space-page-8.png"  width="520px" align="left"/>
<img class="nofancybox" src="/images/div.jpg" width="100%" height="0.1px"/>

注：FREE、FULL 列表中的元素为 XDES Entry。

### INDEX Page

InnoDB 采用了 b+ 树进行数据的管理，数据本身也存在主键中，所以在 Space 内部，所有真实的用户数据都是保存在 INDEX Page 中。

<img src="/images/space-page-9.png"  width="520px" align="left"/>

<img src="/images/space-page-10.png"  width="520px" align="left"/>

<img src="/images/space-page-11.png"  width="520px" align="left"/>

<img src="/images/space-page-12.png"  width="520px" align="left"/>
<img class="nofancybox" src="/images/div.jpg" width="100%" height="0.1px"/>

System Record 的格式如下：

<img src="/images/space-page-13.png"  width="520px" align="left"/>
<img class="nofancybox" src="/images/div.jpg" width="100%" height="0.1px"/>

User Record 的格式如下：

<img src="/images/space-page-14.png"  width="520px" align="left"/>

<img src="/images/space-page-15.png"  width="520px" align="left"/>

<img src="/images/space-page-16.png"  width="520px" align="left"/>

<img src="/images/space-page-17.png"  width="520px" align="left"/>

<img src="/images/space-page-18.png"  width="520px" align="left"/>
<img class="nofancybox" src="/images/div.jpg" width="100%" height="0.1px"/>

## 总体图

<img src="/images/space-page-19.png"  width="780px" align="left"/>

<img src="/images/space-page-20.png"  width="780px" align="left"/>
<img class="nofancybox" src="/images/div.jpg" width="100%" height="0.1px"/>

## 参考文献

> https://blog.jcole.us/2013/01/03/the-basics-of-innodb-space-file-layout/
>
> https://blog.jcole.us/2013/01/04/page-management-in-innodb-space-files/
>
> https://blog.jcole.us/2013/01/07/the-physical-structure-of-innodb-index-pages/
>
> https://blog.jcole.us/2013/01/10/btree-index-structures-in-innodb/
>
> https://blog.jcole.us/2013/01/10/the-physical-structure-of-records-in-innodb/
>
> https://blog.jcole.us/2013/01/14/efficiently-traversing-innodb-btrees-with-the-page-directory/