---
layout: post
title: Getting Started | Pan
date: 2016-03-21
categories: blog
tags: [Android, MVC, MVVM, MVP, Pan]
author: nius
url: http://nius.me
---

[Pan (https://github.com/campusappcn/Pan)](https://github.com/campusappcn/Pan)框架使用起来很简单。采用[Pan](https://github.com/campusappcn/Pan)来编写界面和控制代码，可以和原有的代码完全兼容、并存。

首先，是轻量化的Activity代码，主要通过Pan的工厂方法with，得到ViewModel的实例，绑定ViewModel和Controller到Activity上。工厂方法with有很多重载，也可以传入使用实例化好的对象。

```Java
public class MainActivity extends PanFragmentActivity {

    MainViewModel mMainViewModel;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        mMainViewModel = Pan.with(this, MainViewModel.class)
                .controlledBy(MainController.class)
                .getViewModel()
                .setHelloString("hello Pan!")
                .render();
    }

}
```

How clean! 事实上，常见的Activity中需要实现的onResume等方法，也无需写在这里，可以完全交给Controller。

<!-- more -->

接下来编写具体ViewModel实现类。ViewModel主要完成渲染逻辑，因此ViewModel的成员变量包含两种：

1. 以v作为前缀的View对象
2. 以m作为前缀的ViewModel具体的字段

render方法负责将ViewModel字段渲染到View上 
```Java
@Xml(R.layout.activity_main) //可选，让ViewModel语义更明确。当需要自己实例化新View时必选。
public class MainViewModel extends GeneralViewModel {

    @Bind(R.id.hello) //Butterknife
    Button vHelloTv;

    String mHelloString;

    @Override
    public MainViewModel render() {
        vHelloTv.setText(mHelloString);
        return this;
    }

    public MainViewModel setHelloString(String string) {
        mHelloString = string;
        return this;
    }
}
```

其次，是Controller实现类。当然一个View不必要有Controller，如果不需要监听任何事件的话。
Controller通过泛型参数，与ViewModel实现绑定，可以处理两类事件：
1. 用户交互，通过bindEvents()方法实现，$vm为绑定的ViewModel对象
2. 所处Activity/Fragment的生命周期，通过实现接口（例如，OnResume）进行监听

```Java
public class MainController extends GeneralController<MainViewModel> 
		implements OnResume { //以监听Activity的OnResume事件

    @Override
    protected void bindEvents() {
        $vm.vHelloTv.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Toast.makeText(getActivity(), $vm.mHelloString, Toast.LENGTH_SHORT).show();
            }
        });
    }

    @Override
    public void onResume() {
        Log.d("MainController", "On Resume For " + getActivity());
    }
}
```

对应的布局XML:
```XML
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
              android:orientation="vertical"
              android:layout_width="match_parent"
              android:layout_height="match_parent">

    <Button
        android:id="@+id/hello"
        android:text="Hello World!"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"/>


</LinearLayout>
```
添加Pan的依赖：

```groovy
	repositories {
		//jitpack repository
		maven { url "https://jitpack.io" }
	}

	dependencies {
        compile 'com.github.campusappcn:Pan:0.2.2'
	}
```
