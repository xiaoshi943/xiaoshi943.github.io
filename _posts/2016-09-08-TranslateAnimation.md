## TranslateAnimation 的setAnimation()和startAnimation()

### 1、View. startAnimation()

​	该方法会立即执行动画，控制不住其开始时间。同时，如果在handleMessage()中调用该方法不起作用。

### 2、View. setAnimation()

​	该方法需要调用animation.start()方法才能执行，能控制其在什么时候开始。在handleMessage()中也起作用。

```java
	   TranslateAnimation translateAnimation = 
  					newTranslateAnimation(0.0f, 0.0f,0.0f,-60.0f); 
       translateAnimation.setDuration(500);

       translateAnimation.setAnimationListener(newAnimation.AnimationListener() {

           @Override
           public void onAnimationStart(Animation animation) {

           }

           @Override
           public void onAnimationRepeat(Animation animation) {

           }

           @Override
           public void onAnimationEnd(Animation animation) {

                view.clearAnimation();
                int left = view.getLeft();
                int top =view.getTop()-60;
                int width = view.getWidth();
                int height = view.getHeight();
                view.layout(left, top, left+width, top+height);
           }
       });    
```



 

