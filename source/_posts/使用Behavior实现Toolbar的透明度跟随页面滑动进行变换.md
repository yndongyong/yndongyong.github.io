---
title: 使用Behavior实现Toolbar的透明度跟随页面滑动进行变换
date: 2018-01-11 16:43:07
tags: Android Toolbar Behavior
---

使用Behavior实现Toolbar的透明度跟随页面上下滑动从不透明到透明以及反向的一个显示效果，如下图：

![`Toobar`的显示与隐藏](http://oav23hfp9.bkt.clouddn.com/18-4-27/40610293.jpg)


###一、实现的效果

1. 界面上滑过程中，Toolbar逐渐显示出来，直到完全显示
2. 界面下滑过程中，Toolbar逐渐隐藏，直到完全不可见

###二、常规思路

如图所示的效果，常规的做法是为AppBarLayout添加一个OnOffsetChangedListener来监听appbarlayout的滑动的过程，用当前滑动的距离和slideDistance某个固定值计算一个比例，比如图中红色背景块作为锚点，slideDistance就是这个view的高度，通过比例计算出当前的Toolbar的背景颜色的透明值。

1.上滑过程中滑动距离从 0 到 slideDistance 变化，背景颜色根据滑动距离百分比改变颜色透明度，这个过程颜色透明度从 0 到 255。

2.下滑过程：即上滑过程的逆向。

通常情况下，slideDistance 是某个UI元素的高度，这个高度还需要动态的拿到。一般的做法是在Activity的onCreated方法中使用View.post(runnable),或者添加个一个addOnGlobalLayoutListener监听器待界面绘制完成后得到view的高度。

OnOffsetChangedListener的核心代码如下：

```
AppBarLayout.OnOffsetChangedListener offsetChangedListener = new AppBarLayout.OnOffsetChangedListener() {
        @RequiresApi(api = Build.VERSION_CODES.M)
        @Override
        public void onOffsetChanged(AppBarLayout appBarLayout, int verticalOffset) {
            float fraction = Math.abs(verticalOffset * 1.0f) / slideDistance;
            fraction = Math.min(fraction, 1);
            bgColor = changeAlpha(getResources().getColor(R.color.white), fraction);
            header_toolbar.setBackgroundColor(bgColor);
            tv_toolbar_title.setAlpha(fraction);
        }
    };

```

### 三、优化后的思路

由于项目中的多个详情界面都需要实现这样的效果，造成了代码的多次拷贝，如果将来UI需要改变的话，散落在项目各个角落的代码也将难以维护。此外，在每个界面中Toobar显示隐藏参考的view元素也不同的。

基于以上这些缺点项目最终废弃了常规的做法，使用AppBarLayout.Behavior将这一效果封装、抽象。
在需要使用的地方直接引用自定义的Behavior就行。通过Behavior，以一种非侵入的方式为view添加
行为。

核心代码如下：

```

public class ChangeToolbarAlphaBehavior extends AppBarLayout.Behavior {

    //锚点view
    private static final String TAG_ANCHOR_VIEW = "anchorView";
    //title
    private static final String TAG_TOOLBAR_TITLE = "toolbarTitle";
    //toolbar
    private static final String TAG_TOOLBAR = "toolbar";

    private TextView header_title;
    private View anchorView;
    private android.support.v7.widget.Toolbar toolbar;

    float slideDistance = 0f;

    private int bgColor;

    private int mainColor = 0;

    public ChangeToolbarAlphaBehavior(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    /**
     * AppBarLayout布局时调用
     * @param parent
     * @param abl
     * @param layoutDirection
     * @return
     */
    @Override
    public boolean onLayoutChild(CoordinatorLayout parent, AppBarLayout abl, int layoutDirection) {

        //拿到toolbar
        if (toolbar == null) {
            toolbar = parent.findViewWithTag(TAG_TOOLBAR);
        }
        //拿到title
        if (header_title == null) {
            header_title = parent.findViewWithTag(TAG_TOOLBAR_TITLE);
        }
        //拿到锚点View，以锚点view的高度作为完全显示或者隐藏的临界值
        if (anchorView == null) {
            anchorView = parent.findViewWithTag(TAG_ANCHOR_VIEW);
        }
        if (anchorView == null || toolbar == null) {
            return super.onLayoutChild(parent, abl, layoutDirection);
        }
        //临界值
        if (slideDistance == 0f) {
            slideDistance = anchorView.getBottom() - toolbar.getHeight();
        }
        if (mainColor == 0) {
            mainColor = ContextCompat.getColor(parent.getContext(), R.color.white);
        }

        abl.addOnOffsetChangedListener(new AppBarLayout.OnOffsetChangedListener() {
            @Override
            public void onOffsetChanged(AppBarLayout appBarLayout, int verticalOffset) {
                float fraction = Math.abs(verticalOffset * 1.0f) / slideDistance;
                fraction = Math.min(fraction, 1);

                bgColor = changeAlpha(mainColor, fraction);
                toolbar.setBackgroundColor(bgColor);
                header_title.setAlpha(fraction);
            }
        });

        return super.onLayoutChild(parent, abl, layoutDirection);
    }

    /**
     * 根据百分比改变颜色透明度
     */
    public int changeAlpha(int color, float fraction) {
        int red = Color.red(color);
        int green = Color.green(color);
        int blue = Color.blue(color);
        int alpha = (int) (Color.alpha(color) * fraction);
        return Color.argb(alpha, red, green, blue);
    }
}

```

使用方式

1、直接在xml里为AppbarLayout添加自定义的behavior

`app:layout_behavior="packagename.ChangeToolbarAlphaBehavior"`

2、为toolbar添加tag

` android:tag="toolbar" `

3、 为锚点view添加tag

` android:tag="anchorView" `

3、 为toolbar中有相同动作的view添加tag

比如此处的title
` android:tag="toolbarTitle" `









