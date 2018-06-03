---
title: 使用RecyclerView实现分组的展开与收缩
date: 2018-01-27 18:05:12
tags: Android RecycleView 分组 Section
---


使用RecyclerView实现分组的折叠和展开的显示效果，先上效果图：
![效果图](http://oav23hfp9.bkt.clouddn.com/18-5-7/87674041.jpg)


### 思路
如图中所示，有多个Section，每个Section对应着一组数据。可以直接点击Section或者Section的箭头来展开显示当前Section下的数据，或者收起当前Section。项目中使用RecyclerView实现这种效果。主要的思路是：使用两种viewtype实现对应的Section和下层的Entity布局，当展开和折叠操作触发时动态计算RecyclerView对应的Adapter应该包含的数据集，该数据集包含Section数据集和Entity数据集，之后再触犯RecyclerView刷新。

Adapter的数据集计算规则如下举例:

* Section都是折叠效果的时，Adapter对应的数据集是：list.addAll(所有Section)
* 第一个Section展开时的Adapter对应的数据集是：list.add(第一个Setction),list.addAll(第一个Section下的Entity数据集),list.add(其他Section) 

 
定义Section实体
		
	public class Section {

	    private String id;
	
	    //title
	    private String name;
	    //subtitle
	    private String subTitle;
	    //Section是否是展开状态 true：展开
	    private boolean isExpanded;
	
			<!--其他属性和getter/setter方法-->
	} 

定义第二层的实体ProgramEntity

	public class ProgramEntity {

	    private String id;
	    private String name;
	    private String abumId;
	    private String content;
	    <!--其他属性和getter/setter方法-->

	}

项目中的使用的Adapter是之前的封装的Adapter,其思想同代码家的MultiType

在Adapter中为项目注册类型和布局提供者的关系

	mSectionedExpandableGridAdapter = new ExpandableGridAdapter(context, mDataArrayList);
	<!--对应Section布局-->
        mSectionedExpandableGridAdapter.register(Section.class, new SectionItemViewProvider(sectionStateChangeListener));
	<!--对应Entity布局-->
        ProgramStyle2ItemViewProvider programStyle2ItemViewProvider
                = new ProgramStyle2ItemViewProvider();
        mSectionedExpandableGridAdapter.register(ProgramEntity.class, programStyle2ItemViewProvider);
        recyclerView.setAdapter(mSectionedExpandableGridAdapter);
        
当Section被点击是时会回调SectionStateChangeListener，在回调方法中重新计算Adapter数据集中应包含的数据，在触发Recyclerview的刷新。

核心代码

	//data list
    private LinkedHashMap<Section, List<ProgramEntity>> mSectionDataMap = new LinkedHashMap<Section, List<ProgramEntity>>();
    private Items mDataArrayList = new Items();
    
    public void notifyDataSetChanged() {
        //先清空原本的数据
        mDataArrayList.clear();
        for (Map.Entry<Section, List<ProgramEntity>> entry : mSectionDataMap.entrySet()) {
            Section key;
            //将Section添加到数据容器中
            mDataArrayList.add((key = entry.getKey()));
            //Section是展开状态将Section对应的Entity数据集添加到数据容器中
            if (key.isExpanded())
                mDataArrayList.addAll(entry.getValue());
        }
        mSectionedExpandableGridAdapter.notifyDataSetChanged();
    }
          
如果Section下的数据展示是多列，Recyclerview还可以使用GridLayoutManager，通过SpanSizeLookup控制Section单独显示一行，而二层数据显示为多列。

	gridLayoutManager.setSpanSizeLookup(new GridLayoutManager.SpanSizeLookup() {
            @Override
            public int getSpanSize(int i) {
                return !isSection(i)?gridLayoutManager.getSpanCount():1;
            }
        });
        
如图所示的效果，仅为演示，UI就不再单独处理。

![Section单独占一行，二级为gride的效果图](http://oav23hfp9.bkt.clouddn.com/18-5-7/15510048.jpg)

