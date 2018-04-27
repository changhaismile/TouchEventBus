# 滑动冲突解决方案

---
# 非嵌套滑动

Android原生的触摸事件分发总是从父布局开始分发，从最顶层的子View开始处理，这种特性有时候会限制了我们一些很复杂的交互设计。
``TouchEventBus`` 致力于解决非嵌套的滑动冲突，比如多个 **在同一层级** 的``Fragment`` 对触摸事件的处理。触摸事件会先到达顶层 ``Fragment`` 的 ``onTouch`` 方法，然后逐层判断是否消费，在都不消费的情况下才到底层的 ``Fragment`` 。而且这些层级互不嵌套，没有形成parent和child的关系，意味着想通过 ``onInterceptTouchEvent`` 或者 ``requestDisallowInterceptTouchEvent`` 方法来调整事件分发都是不可能的。

## 同级视图的触摸事件

下面是手机YY的开播预览页：

![YY预览页][1]

在这个页面上有很多对触摸事件的处理，包括且不限于：

- 在屏幕上点击，会触发摄像头的聚焦（黄色框出现的地方）
- 双指缩放，会触发摄像头的缩放
- 左右滑动，可以切换 ``ViewPager`` ，从“直播”和“玩游戏”两个选项卡之间切换
- “玩游戏”选项卡上的列表可以滑动
- “直播”选项卡上的控件可以点击（开播按钮，添加图片…）
- 由于预览页和开播页是同一个 ``Activity`` ，所以这个 ``Activity`` 上还有很多开播后的 ``Fragment``,比如公屏等等也有触摸事件

从视觉上可以判断出View Tree的层级以及对触摸处理的层级：

![处理顺序][2]

左边的是View的层级，上层是 ``ViewPager`` 以及上面的View，下面是显示视频流的 ``Fragment``。右边是触摸事件处理的层级，双指缩放/View点击/聚焦点击需要在 ``ViewPager``上面，否则都会被 ``ViewPager`` 消费掉，但是 ``ViewPager`` 的View层级又比视频 ``Fragment`` 要高。这就是非嵌套的滑动冲突的核心矛盾：

>> **业务逻辑的层级** 与 **用户看到的UI层级** 顺序不一致

## 对触摸事件的重新分发

手机YY直播间中的 ``Fragment``  非常多，而且因为插件化的原因，各个业务插件可以动态地往直播间添加/移除自己业务的 ``Fragment`` ，这些 ``Fragment`` 层级相同互不嵌套，有自己比较独立的业务逻辑，也会有点击/滑动等事件处理的需求。但由于业务场景复杂，``Fragment`` 的上下层级顺序也会动态改变，这就很容易导致一些 ``Fragment`` 一直收不到触摸事件或者在切换业务模板的时候触摸事件被其他业务消费。

``TouchEventBus`` 用于这种场景下对触摸事件进行重新分发，我们可以随心所欲地决定业务逻辑的层级顺序。

![TouchEventBus重新分发触摸事件][3]

每个手势的处理就是一个 ``TouchEventHandler``，比如镜头的缩放是 **CameraZoomHandler** ，镜头的聚焦点击是 **CameraClickHandler** ，``ViewPager`` 滑动是 **PreviewSlideHandler** ，然后重新为这些 Handler 作一个排序，按照业务的需要来传递 ``MotionEvent`` 。然后 ``TouchEventHandler`` 需要和ui对应起来，通过Handler的 ``attach`` / ``dettach`` 方法来指定对应的ui。而ui可以是一个具体的 ``Fragment``，也可以是一个抽象的接口，代表一个对触摸事件作出响应的业务。

比如在开播预览页的聚焦点击处理，先是定义ui的接口：

```Java
public interface CameraClickView {
    /**
     * 在指定位置为中心显示一个黄色矩形的聚焦框
     *
     * @param x 手指触摸坐标x
     * @param y 手指触摸坐标y
     */
    void showVideoClickFocus(float x, float y);

    /**
     * 给VideoSdk传递触摸事件，让其在指定坐标进行摄像头聚焦
     *
     * @param e 触摸事件
     */
    void onTouch(MotionEvent e);
}
```

然后是 ``TouchEventHandler`` 的定义：

```Java
public class CameraClickHandler extends TouchEventHandler<CameraClickView> {
    
    private boolean performClick = false;
    //...
    
    @Override
    public boolean onTouch(@NonNull CameraClickView v, MotionEvent e, boolean hasBeenIntercepted) {
        super.onTouch(v, e, hasBeenIntercepted);
        if (!isCameraFocusEnable()) { //一些特殊业务需要禁止摄像头聚焦
            return false;
        }
        //通过MotionEvent判断performClick是否为true
        switch (e.getAction()) {
            case MotionEvent.ACTION_DOWN:
                //...
                break;
            case MotionEvent.ACTION_MOVE:
                //...
                break;
            case MotionEvent.ACTION_UP: 
                //...
                break;
            default:
                break;
        }

        if (performClick) { //认为是点击行为，调用ui的接口
            v.showVideoClickFocus(e.getRawX(), e.getRawY());
            v.onTouch(e);
        }
        return performClick; //点击的时候消费掉触摸事件
    }
}
```

最后是 ``TouchEventHandler`` 与 ui 的对应的绑定

```Java
public class MobileLiveVideoComponent extends Fragment implements CameraClickView{
    
    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {
        //...
        //CameraClickHandler与当前Fragment绑定
        TouchEventBus.of(CameraClickHandler.class).attach(this);
    }
    
    @Override
    public void onDestroyView() {
        //...
        //CameraClickHandler与当前Fragment解绑
        TouchEventBus.of(CameraClickHandler.class).dettach(this);
    }
    
    @Override
    public void showVideoClickFocus(float x, float y) {
        //todo: 展示一个黄色框ui
    }
    
    @Override
    public void onTouch(MotionEvent e) {
        //todo: 调用SDK的摄像头聚焦
    }
}
```

当用户对ui的进行手势操作时，``MotionEvent`` 就会沿着 ``TouchEventBus`` 里面的顺序进行分发。如果在 **CameraClickHandler** 之前没有别的 Handler 把事件消费掉，那么就能在 ``onTouch`` 方法进行处理，然后在ui有聚焦的响应。


  [1]: https://github.com/YvesCheung/TouchEventBus/blob/master/img/touchEventBusInYYPreview.gif
  [2]: https://raw.githubusercontent.com/YvesCheung/TouchEventBus/master/img/touchOrder.png
  [3]: https://raw.githubusercontent.com/YvesCheung/TouchEventBus/master/img/TouchEventBus.png
