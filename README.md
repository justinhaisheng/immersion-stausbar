# immersion-stausbar 优雅的实现沉浸式状态栏

>首先来说，从Android 4.4开始才能实现沉浸式状态栏的，所以如果您的APP也支持Android 4.4以下版本，那还需要对Android 4.4以下版本做“不支持沉浸式状态栏”处理。那么，从Android 4.4开始，大概可以分成三个阶段来实现沉浸式状态栏：


- Android4.4(API 19) - Android 5.0(API 21)：这个阶段的实现方式为：通过 **FLAG_TRANSLUCENT_STATUS**设置状态栏为透明并且为全屏模式，然后通过添加一个与StatusBar一样大小的View，将View的背景设置为要设置的颜色，从而实现沉浸式。
- Android 5.0(API 21) - Android 6.0(API 23)： 从Android 5.0开始，加入了一个重要的属性**android:statusBarColor**和方法**setStatusBarColor()**，通过这个方法我们就可以轻松实现沉浸式状态栏。但是在`Android 6.0`以下版本官方不支持设置状态栏的文字和图标颜色，目前只有小米和魅族的ROM提供了支持。
- Android 6.0(API 23)以上版本：其实`Android 6.0`以上的实现方式和`Android 5.0+` 是一样的，区别是`从Android 6.0`开始，官方支持改变状态栏的文字和图标的颜色。

#方案

##修改应用主题
* 为了更好地演示沉浸式状态栏的效果，我们修改应用的主题，使其不要显示Android默认的标题栏。打开styles.xml文件，可以改成如下代码，并且将colorPrimary等颜色设置删掉。


```xml

<style name="AppTheme" parent="Theme.AppCompat.Light.NoActionBar"></style>

```

* 获取状态栏高度

```java

public class StatusBarUtils {
    public static int getHeight(Context context) {
        int statusBarHeight = 0;
        try {
            int resourceId = context.getResources().getIdentifier("status_bar_height", "dimen",
                    "android");
            if (resourceId > 0) {
                statusBarHeight = context.getResources().getDimensionPixelSize(resourceId);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return statusBarHeight;
    }
}

```

* Android 5.0+

```java

public static void setColor(@NonNull Window window, @ColorInt int color) {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
        window.addFlags(WindowManager.LayoutParams.FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS);
        window.clearFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS);
        window.clearFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_NAVIGATION);
        window.getDecorView().setSystemUiVisibility(View.SYSTEM_UI_FLAG_LAYOUT_STABLE);
        window.setStatusBarColor(color);
    }
}

```
修改状态栏颜色的功能其实就是对Window进行操作，而该Window可以是Activity或Dialog等持有的Window，所以我们就封装了一个传递Window的方法。
为了便于对Activity直接操作，可以再增加一个如下方法

```java

public static void setColor(Context context, @ColorInt int color) {
    if (context instanceof Activity) {
        setColor(((Activity) context).getWindow(), color);
    }
}

```

* 官方也仅支持设置状态栏文字和图标的深色模式和浅色模式，但是官方仅在Android 6.0以上版本提供支持。设置代码如下：


```java
private static void setTextDark(Window window, boolean isDark) {
    if (android.os.Build.VERSION.SDK_INT >= android.os.Build.VERSION_CODES.M) {
        View decorView = window.getDecorView();
        int systemUiVisibility = decorView.getSystemUiVisibility();
        if (isDark) {
            decorView.setSystemUiVisibility(systemUiVisibility | View.SYSTEM_UI_FLAG_LIGHT_STATUS_BAR);
        } else {
            decorView.setSystemUiVisibility(systemUiVisibility & ~View.SYSTEM_UI_FLAG_LIGHT_STATUS_BAR);
        }
    }
}

```

同样再增加一个对Activity的支持：

```java

public static void setTextDark(Context context, boolean isDark) {
    if (context instanceof Activity) {
        setTextDark(((Activity) context).getWindow(), isDark);
    }
}

```

为了能够根据状态栏背景颜色的深浅而自动设置文字的颜色，我们再新增一个判断颜色深浅的方法：


```java


public static boolean isDarkColor(@ColorInt int color) {
    return ColorUtils.calculateLuminance(color) < 0.5;
}


```

然后在setColor()方法里新增一行设置状态栏文字颜色的代码，如下：

```java

public static void setColor(@NonNull Window window, @ColorInt int color) {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
        window.addFlags(WindowManager.LayoutParams.FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS);
        window.clearFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS);
        window.clearFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_NAVIGATION);
        window.getDecorView().setSystemUiVisibility(View.SYSTEM_UI_FLAG_LAYOUT_STABLE);
        window.setStatusBarColor(color);
        setTextDark(window, !isDarkColor(color));
    }
}


```
上面是Android 6.0以上版本的实现，那么对于Android 6.0以下的手机怎么办呢？目前Android 5.0-6.0的手机只有小米MIUI和魅族Flyme系统提供了支持。小米MIUI的设置方法如下：

```java

private static void setMIUIDark(Window window, boolean isDark) {
    try {
        Class<? extends Window> clazz = window.getClass();
        int darkModeFlag;
        Class<?> layoutParams = Class.forName("android.view.MiuiWindowManager$LayoutParams");
        Field field = layoutParams.getField("EXTRA_FLAG_STATUS_BAR_DARK_MODE");
        darkModeFlag = field.getInt(layoutParams);
        Method extraFlagField = clazz.getMethod("setExtraFlags", int.class, int.class);
        extraFlagField.invoke(window, isDark ? darkModeFlag : 0, darkModeFlag);
    } catch (Exception e) {
        e.printStackTrace();
    }
}


```

魅族Flyme的设置方法如下：

```java
private static void setFlymeDark(Window window, boolean isDark) {
    if (window != null) {
        try {
            WindowManager.LayoutParams lp = window.getAttributes();
            Field darkFlag = WindowManager.LayoutParams.class
                    .getDeclaredField("MEIZU_FLAG_DARK_STATUS_BAR_ICON");
            Field meizuFlags = WindowManager.LayoutParams.class
                    .getDeclaredField("meizuFlags");
            darkFlag.setAccessible(true);
            meizuFlags.setAccessible(true);
            int bit = darkFlag.getInt(null);
            int value = meizuFlags.getInt(lp);
            if (isDark) {
                value |= bit;
            } else {
                value &= ~bit;
            }
            meizuFlags.setInt(lp, value);
            window.setAttributes(lp);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
然后在setTextDark()方法中添加如下代码：
```java

private static void setTextDark(Window window, boolean isDark) {
    if (android.os.Build.VERSION.SDK_INT >= android.os.Build.VERSION_CODES.M) {
        View decorView = window.getDecorView();
        int systemUiVisibility = decorView.getSystemUiVisibility();
        if (isDark) {
            decorView.setSystemUiVisibility(systemUiVisibility | View.SYSTEM_UI_FLAG_LIGHT_STATUS_BAR);
        } else {
            decorView.setSystemUiVisibility(systemUiVisibility & ~View.SYSTEM_UI_FLAG_LIGHT_STATUS_BAR);
        }
    } else if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
        switch (OSUtils.getRomType()) {
            case MIUI:
                setMIUIDark(window, isDark);
                break;
            case Flyme:
                setFlymeDark(window, isDark);
                break;
            default:
        }
    }
}


```

##实现状态栏透明

>这时一般我们需要将图片顶到状态栏里，也就是整个内容布局顶到状态栏里，并设置状态栏的颜色透明，才能实现沉浸式状态栏的效果。那么，在我们的StatusBarUtils类里添加如下代码：

```java

public static void setTransparent(@NonNull Window window) {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
        window.addFlags(WindowManager.LayoutParams.FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS);
        window.clearFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS);
        window.clearFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_NAVIGATION);
        window.getDecorView().setSystemUiVisibility(View.SYSTEM_UI_FLAG_LAYOUT_STABLE | View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN);
        window.setStatusBarColor(Color.TRANSPARENT);
    }
}


```

同样针对Activity，增加如下方法

```java
public static void setTransparent(Context context) {
    if (context instanceof Activity) {
        setTransparent(((Activity) context).getWindow());
    }
}

```

 # Android 4.4+

下面针对`Android 4.4-5.0`的手机进行实现。实现原理是将内容布局设为全屏，然后在布局的顶部添加一个和状态栏一样高度的View，将该View的背景设置成我们想要的颜色。当需要将状态栏设置纯颜色时，为了`和Android5.0`以上版本保持一致，我们对内容布局的上边设置一个`padding`，大小为状态栏的高度。

为了能复用这个`View`，我们新增一个自定义的ID，在values文件夹下新建`ids.xml`文件，新增代码如下：

```java
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <item name="fake_status_bar_view" type="id"/>
</resources>
```

然后在StatusBarUtils类里添加如下代码：

```
private static final int FAKE_STATUS_BAR_VIEW_ID = R.id.fake_status_bar_view;
 
@RequiresApi(api = Build.VERSION_CODES.KITKAT)
public static void setColor(@NonNull Window window, @ColorInt int color,
                            boolean isTransparent) {
    Context context = window.getContext();
    window.addFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS);
    window.clearFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_NAVIGATION);
    ViewGroup decorView = (ViewGroup) window.getDecorView();
    View contentView = decorView.findViewById(android.R.id.content);
    if (contentView != null) {
        contentView.setPadding(0, isTransparent ? 0 : getHeight(context), 0, 0);
    }
    View fakeStatusBarView = decorView.findViewById(FAKE_STATUS_BAR_VIEW_ID);
    if (fakeStatusBarView != null) {
        fakeStatusBarView.setBackgroundColor(color);
        if (fakeStatusBarView.getVisibility() == View.GONE) {
            fakeStatusBarView.setVisibility(View.VISIBLE);
        }
    } else {
        // 绘制一个和状态栏一样高的矩形
        View statusBarView = new View(context);
        FrameLayout.LayoutParams layoutParams =
                new FrameLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT,
                        getHeight(context));
        statusBarView.setLayoutParams(layoutParams);
        statusBarView.setBackgroundColor(color);
        statusBarView.setId(FAKE_STATUS_BAR_VIEW_ID);
        decorView.addView(statusBarView);
    }
}

```

在设置纯颜色时，我们还需要将该颜色与黑色进行1:1的混合。为什么要这么设置呢？因为状态栏的文字和图标颜色默认是白色的，并且在Android 5.0以下是不能修改的，所以如果修改成较浅的颜色，就会导致状态栏文字看不清的现象，因此做一个比较暗的浮层效果更好一些。那么将setColor()方法改成如下代码：

```java

public static void setColor(@NonNull Window window, @ColorInt int color) {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
        window.addFlags(WindowManager.LayoutParams.FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS);
        window.clearFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS);
        window.clearFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_NAVIGATION);
        window.getDecorView().setSystemUiVisibility(View.SYSTEM_UI_FLAG_LAYOUT_STABLE);
        window.setStatusBarColor(color);
        setTextDark(window, !isDarkColor(color));
    } else if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
        setColor(window, ColorUtils.blendARGB(Color.TRANSPARENT, color, 0.5f), false);
    }
}

```

设置状态栏透明

在设置状态栏透明时，为了也能清楚地看清状态栏的文字，我们直接设置状态栏的颜色为50%透明度的黑色。

于是，修改setTransparent()方法如下：

```ava
public static void setTransparent(@NonNull Window window) {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
        window.addFlags(WindowManager.LayoutParams.FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS);
        window.clearFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS);
        window.clearFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_NAVIGATION);
        window.getDecorView().setSystemUiVisibility(View.SYSTEM_UI_FLAG_LAYOUT_STABLE | View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN);
        window.setStatusBarColor(Color.TRANSPARENT);
    } else if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
        setColor(window, 0x80000000, true);
    }
}


```


# 特殊场景（启动页、刘海屏）

## 启动页

对于启动页，一般都会把状态栏收上去，这需要适配刘海屏，否则刘海区域会显示黑的一片。其实Android P以上提供了适配刘海屏的方法，在启动页Activity添加如下代码：
```
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_splash);
    StatusBarUtils.setTransparent(this);
    // 适配刘海屏
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.P) {
        WindowManager.LayoutParams layoutParams = getWindow().getAttributes();
        layoutParams.layoutInDisplayCutoutMode =
                WindowManager.LayoutParams.LAYOUT_IN_DISPLAY_CUTOUT_MODE_SHORT_EDGES;
        getWindow().setAttributes(layoutParams);
    }
}

```

但是一些手机厂商的刘海屏手机系统版本是低于Android P的，不过也都提供了适配的方法。适配方式是在AndroidManifest.xml文件里的application标签下添加如下代码：

```
<!-- 允许绘制到小米刘海屏机型的刘海区域 -->
<meta-data
    android:name="notch.config"
    android:value="portrait" />
<!-- 允许绘制到华为刘海屏机型的刘海区域 -->
<meta-data
    android:name="android.notch_support"
    android:value="true" />
<!-- 允许绘制到oppo、vivo刘海屏机型的刘海区域 -->
<meta-data
    android:name="android.max_aspect"
    android:value="2.2" />


```

另外，对于Android 5.0以下的手机，适配完刘海屏后会在顶部多一块黑色半透明的View，那我们将其改成全透明的，修改onCreate()方法如下：

```java

@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_splash);
    StatusBarUtils.setTransparent(this);
    // 适配刘海屏
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.P) {
        WindowManager.LayoutParams layoutParams = getWindow().getAttributes();
        layoutParams.layoutInDisplayCutoutMode =
                WindowManager.LayoutParams.LAYOUT_IN_DISPLAY_CUTOUT_MODE_SHORT_EDGES;
        getWindow().setAttributes(layoutParams);
    }
    // 适配Android 4.4
    if (Build.VERSION.SDK_INT < Build.VERSION_CODES.LOLLIPOP) {
        StatusBarUtils.setColor(getWindow(), Color.TRANSPARENT, true);
    }
}

```

## 参考
* [从0到1优雅实现沉浸式状态栏](https://blog.csdn.net/u013541140/article/details/100065336)
* [MIUI 9 & 10“状态栏黑色字符”实现方法变更通知](https://dev.mi.com/console/doc/detail?pId=1159)
* [简洁明了的刘海屏适配方案](https://www.jianshu.com/p/c9e710a9fa35)