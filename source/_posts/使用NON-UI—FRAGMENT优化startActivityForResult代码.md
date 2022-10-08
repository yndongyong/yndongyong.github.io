---
title: 使用NON-UI—FRAGMENT优化startActivityForResult代码
date: 2018-08-30 10:30:58
tags: non-ui-framgment headlessfragment 
categories: 
- android
---

## 使用NON-UI-FRAGMENT优化startActivityForResult

一般地，要使用AActivity.startActivityForResult(BActivity)需要在Activity中覆盖父类的onActivityForResult（）回调方法。

如果一个Activity需要调用多个Activity并且获取返回结果，会造成调用处和回调处彼此分离，逻辑不清晰、代码混乱的的问题。有没有一种较为优雅的方式，打破这种惯性。

回答是肯定的，即本文的NON-UI-FRAGMENT，使用无UI的fragment，将部分业务代码封装到fragment中，再通过callback将最终结果回调给Activity，原本的Activity再也不用复写父类的startActivityForResult，传递一个回调即可。

使用这种方式，可以封装那些共性较多的类，将原本需要通过继承和复写的逻辑放到fragment中，体现了软件开发中的一大设计原则——使用组合替代继承。

本文中的startActivityForResult的替代优化仅是NON-UI-FRAGMENT的一种实现使用，NON-UI-FRAGMENT的可以做的更多，一些简单的使用场景：譬如封装权限申请、登录判断跳转，第三方登录等、截屏等等，其他待自己拓展。

### 使用方式


	ActivityResultHelper resultHelper = ActivityResultHelper.attach(this);
	resultHelper.startForResult(intent, 0x10, new ActivityResultHelper.CallBack() {
	    @Override
	    public void onActivityResult(int requestCode, int resultCode, Intent data) {
	        if (requestCode == 101 && resultCode == RESULT_OK) {
	            Log.d("DONG", "true");
	        } else {
	            Log.d("DONG", "false");
	        }
	    }
	});

再也不用将调用入口和返回结果处理的代码分开了。 

### 完整代码

	 public class ActivityResultHelper extends Fragment {
	
	    private static final String FRAG_TAG = ActivityResultHelper.class.getCanonicalName();
	
	    private Activity mContext;
	
	    private CallBack mListener;
	
	    public interface CallBack {
	        void onActivityResult(int requestCode, int resultCode, Intent data);
	    }
	
	    public static <ParentFrag extends Fragment> ActivityResultHelper attach(ParentFrag parent) {
	        return attach(parent.getChildFragmentManager());
	    }
	
	    public static <ParentActivity extends FragmentActivity> ActivityResultHelper attach(ParentActivity parent) {
	        return attach(parent.getSupportFragmentManager());
	    }
	
	    private static ActivityResultHelper attach(FragmentManager fragmentManager) {
	        ActivityResultHelper frag = (ActivityResultHelper) fragmentManager.findFragmentByTag(FRAG_TAG);
	        if (frag == null) {
	            frag = new ActivityResultHelper();
	            fragmentManager.beginTransaction().add(frag, FRAG_TAG).commit();
	            //TODO fragment在Activity的onCreate中被attach之后就立即调用fragment的一些方法，需要如下代码，否则不需要
	            fragmentManager.executePendingTransactions();
	        }
	        return frag;
	    }
	
	    @Override
	    public void onAttach(Context context) {
	        super.onAttach(context);
	        mContext = (Activity) context;
	    }
	
	    @Override
	    public void onDetach() {
	        super.onDetach();
	        mContext = null;
	    }
	
	    public Activity getContext() {
	        return mContext;
	    }
	
	    public void startForResult(Intent intent, int requestCode, CallBack listener) {
	        this.mListener = listener;
	        startActivityForResult(intent, requestCode);
	    }
	
	    @Override
	    public void onActivityResult(int requestCode, int resultCode, Intent data) {
	        if (mListener != null) {
	            mListener.onActivityResult(requestCode, resultCode, data);
	        }
	    }
	}      

