---
title: RecyclerView 通用Adapter支持多种布局
date: 2018-06-03 12:43:16
tags: RecyclerView Adapter
---

库地址：https://github.com/yndongyong/multiitemview/tree/ontomany_0.0.3

作用：简化adapter中涉及多种ItemView布局的写法，使用对象池存放布局对象(ItemViewProvider)，每一个布局委托一个ItemViewProvider，并在其中封装布局与对象绑定相关的逻辑，简化Adapter。

### 特性

- 极少的类文件
- 简化Adapter的写法，直接使用SimpleAdapter
- 支持多种布局类型，扩展方便
- 每一种布局相关逻辑均写在对应的委托类中
- 支持一种数据类型对应多种布局


### 主要涉及的类

#### SimpleAdapter
继承RecylerView的adapter，简化多类型Item的Adapter写法。

#### ItemViewProvider

针对每一种布局Item的委托类，onCreateViewHolder和onBindViewHolder都由其委托，在其中完成布局的解析和对象的绑定和该布局相关的逻辑，向外暴露两个方法，一个是返回布局id,一个用于ViewHolder和实体对象的绑定。

	int getLayoutId()//布局文件id
	void onBindViewHolder(SimpleViewHolder holder, T entity);//绑定UI和值

#### Items 

用于存放数据集


### 实现思想

- 使用一个SparseArray来保存viewType(即是getItemViewType的返回值)和ItemViewProvider的对应关系
- 使用一个SparseArray来保存viewType和Model（实体类型）的对应关系

对应关系举例如下表所示：

|    viewType   |      Model      |   ItemViewProvider   |
|:-------------:|:---------------:| :--------------------|
|       0       |    Category     | CategoryStyle1ItemViewProvider  |
|       1       |    News         | NewsItemViewProvider            |
|       2       |    Category     | CategoryStyle2ItemViewProvider  |


Adapter的getItemViewType方法的思路举例如下：

对应的实体类型是News，先找到他的viewType，为1，在通过viewType找到对应的ItemViewProvider,对应的ItemViewProvider只有一个，getItemViewType则立即放回ViewType

对应的实体类型是Category，先找到他的viewType为0，2，再通过viewType找到的对应的ItemViewViewProvider为多个，最终通过ItemViewProvider的accept方法判断由哪个具体的ItemViewViewProvider处理，最中返回其viewType的值。 
 
### 用法
	//准备数据
	 items.add("头部1 -- type1");
    items.add(new CategoryEntry("http://pic.58pic.com/58pic/14/27/45/71r58PICmDM_1024.jpg", "风景图片1"));
    items.add(new CategoryEntry("http://img0.imgtn.bdimg.com/it/u=1610953019,3012342313&fm=214&gp=0.jpg", "风景图片2"));

    items.add("头部2 -- type2");
    items.add(new CategoryEntry("http://pic.58pic.com/58pic/14/27/45/71r58PICmDM_1024.jpg", "风景图片4", 2));
    items.add(new CategoryEntry("http://img0.imgtn.bdimg.com/it/u=1610953019,3012342313&fm=214&gp=0.jpg", "风景图片5", 2));

	multiTypeAdapter = SimpleAdapter.create(this)
                .addNewData(items)
                .register(new HeaderItemViewProvider())
                .register(new Category1EntryItemViewProvider())
                .register(new Category2EntryItemViewProvider())
                .register(new Category3EntryItemViewProvider())
                .attachToRecyclerView(rv_list);

效果图中一共实现了4中布局类型：

![多种布局](http://oav23hfp9.bkt.clouddn.com/18-6-3/25785972.jpg)

