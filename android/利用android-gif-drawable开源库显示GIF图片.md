#表情制作

##一、前言 
　　[android-gif-drawable](https://github.com/koral--/android-gif-drawable)是一个在Android显示gif图片的开源库，加载大的gif图片时不会出现OOM问题。


### 1. Drawable.Callback接口

 　　Drawable.Callback 一般用于Drawable动画的回调控制，所有的Drawable子类都应该支持这个类，否则动画将无法在View上正常工作（View是实现了这个接口与BackgroundDrawable进行交互的）。像这篇文章就是分析点击效果（selector）的原理-----[Android Drawable 分析](https://www.zybuluo.com/linux1s1s/note/93075)。

　　值得注意的是，Drawable的setCallback保存的Callback是弱引用，因此传进去的Callback对象不能是局部变量，不然Callback对象很快就会被回收导致动画无法继续播放。

```java
/*如果你想实现一个扩展子Drawable的动画drawable，那么你可以通过setCallBack(android.graphics.drawable.Drawable.Callback)来把你实现的该接口注册到动画drawable 
*中。可以实现对动画的调度和执行 
*/   
public static interface Callback {  
        /** 
         * 当drawable重画时触发，这个点上drawable将被置为不可用（起码drawable展示部分不可用） 
         * @param 要求重画的drawable 
         */  
        public void invalidateDrawable(Drawable who);  
  
        /** 
         * drawable可以通过该方法来安排动画的下一帧。可以仅仅简单的调用postAtTime(Runnable, Object, long)来实现该方法。参数分别与方法的参数对 
         *应 
         * @param who The drawable being scheduled. 
         * @param what The action to execute. 
         * @param when The time (in milliseconds) to run 
         */  
        public void scheduleDrawable(Drawable who, Runnable what, long when);  
  
        /** 
         *可以用于取消先前通过scheduleDrawable(Drawable who, Runnable what, long when)调度的某一帧。可以通过调用removeCallbacks(Runnable,Object)来实现 
         * @param who The drawable being unscheduled. 
         * @param what The action being unscheduled. 
         */  
        public void unscheduleDrawable(Drawable who, Runnable what);  
    }  
```

### 2. gif-drawable控件
　　gif-Drawable一共提供了3中可以显示动态图片的控件：GifImageView 、GifImageButton和GifTextView。当需要赋的图像值是gif格式的图片的时候，会显示动态图片，如果是普通的静态图片，例如是png,jpg的，这个时候，gifImageView等这些控件的效果和ImageView是一样的，也就是说gif-drawable比ImageView更强大，使用的时候跟一般的控件一样。

```xml
<pl.droidsonroids.gif.GifImageView
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:src="@drawable/src_anim"
    android:background="@drawable/bg_anim"
    />
```


###3. GifDrawable类
　　GifDrawable是该开源库中最重要的一个类，GifDrawable可以通过以下方式构建GifDrawable对象
```java
       //asset file
        GifDrawable gifFromAssets = new GifDrawable( getAssets(), "anim.gif" );

        //resource (drawable or raw)
        GifDrawable gifFromResource = new GifDrawable( getResources(), R.drawable.anim );

        //byte array
        byte[] rawGifBytes = ...
        GifDrawable gifFromBytes = new GifDrawable( rawGifBytes );

        //FileDescriptor
        FileDescriptor fd = new RandomAccessFile( "/path/anim.gif", "r" ).getFD();
        GifDrawable gifFromFd = new GifDrawable( fd );

        //file path
        GifDrawable gifFromPath = new GifDrawable( "/path/anim.gif" );

        //file
        File gifFile = new File(getFilesDir(),"anim.gif");
        GifDrawable gifFromFile = new GifDrawable(gifFile);

        //AssetFileDescriptor
        AssetFileDescriptor afd = getAssets().openFd( "anim.gif" );
        GifDrawable gifFromAfd = new GifDrawable( afd );

        //InputStream (it must support marking)
        InputStream sourceIs = ...
        BufferedInputStream bis = new BufferedInputStream( sourceIs, GIF_LENGTH );
        GifDrawable gifFromStream = new GifDrawable( bis );

        //direct ByteBuffer
        ByteBuffer rawGifBytes = ...
        GifDrawable gifFromBytes = new GifDrawable( rawGifBytes );
```

　　GifDrawable实现了Animatable和MediaPlayerControl接口，因此可以通过很多方法对动画进行控制：
```
 - stop() - stops the animation, can be called from any thread
 - start() - starts the animation, can be called from any thread
 - isRunning() - returns whether animation is currently running or not
 - reset() - rewinds the animation, does not restart stopped one
 - setSpeed(float factor) - sets new animation speed factor, eg. passing 2.0f will double the animation speed
 - seekTo(int position) - seeks animation (within current loop) to given position (in milliseconds) Only seeking forward is  - supported
 - getDuration() - returns duration of one loop of the animation
 - getCurrentPosition() - returns elapsed time from the beginning of a current loop of animation
```
　　

　　在GifDrawable里面也可以获取Gif动态图片的一些相关信息
```
 - getLoopCount() - returns a loop count as defined in NETSCAPE 2.0 extension
 - getNumberOfFrames() - returns number of frames (at least 1)
 - getComment() - returns comment text (null if GIF has no comment)
 - getFrameByteCount() - returns minimum number of bytes that can be used to store pixels of the single frame
 - getAllocationByteCount() - returns size (in bytes) of the allocated memory used to store pixels of given GifDrawable
 - getInputSourceByteCount() - returns length (in bytes) of the backing input data
 - toString() - returns human readable information about image size and number of frames (intended for debugging purpose)
 ```

##二、动态表情制作
　　要想高效地展示动态表情，需要考虑以下两方面：
 -  表情的匹配和表情的复用
 -  一个TextView里面可能会有很多表情，应该由具有什么特征的表情对该TextView进行刷新操作
 
###表情的匹配和表情的复用
　　从服务器获取到的表情一般表示 [大笑] ，因此表情的匹配可以利用正则表达式进行匹配，从而找到正确的表情对象GifDrawable，然后作为Spanable中的ImageSpan在TextView进行播放。

　　但是每一次表情匹配都不可能新生成一个GifDrawable对象，这样会很浪费内存，甚至会出现OOM的问题，因此我们需要对表情进行复用，一旦匹配到的表情在之前已经匹配到了，则可以直接拿来复用。

　　下面写一个简单的类对表情进行管理，利用ConcurrentHashMap保存已经生成的表情，每次获取表情都ConcurrentHashMap获取，如果没有就再生成放到ConcurrentHashMap，可以看到保存GifDrawable是一个弱引用，这是保证一旦表情没有任何引用，系统可以尽快回收表情占用的内存。
```java
public class EmotionManager {

    private static final int[] EMOTIONS_IDS = {};    //存储表情的Drawable ID

    private static final String[] EMOTIONS_EXPRESSES = {"[大笑]"};  //表情对应的含义



    private static ConcurrentHashMap<String, WeakReference<GifDrawable>> emotionCacheMap = null; //表情复用


    private static GifDrawable getEmotion(Resources resources, String expression) {
        if(emotionCacheMap == null) {
            emotionCacheMap = new ConcurrentHashMap<>();
        }

        WeakReference<GifDrawable> reference = emotionCacheMap.get(expression);
        if(reference != null && reference.get() != null) {
            return reference.get();
        } else {
            //新生成GifDrawable并放进emotionCacheMap
        }
    }
}

```
###表情的刷新
　　一个表情播放到下一帧的时候需要通知拥有它的TextView进行刷新，根据上面对Drawable.Callback接口的分析，我们可以调用每一个GifDrawable的setCallback函数对该GifDrawable存在的每一个TextView进行刷新。

　　但是该方案存在一个弊端，那就是如果一个TextView里面有很多表情则会导致该TextView刷新过快，最好的方法就是让TextView中刷新频率最高的表情去通知TextView进行刷新，在GifDrawable没有直接获取频率的接口，但是通过getDuration()和getFrameByteCount()可以计算频率。

　　从上面的分析可知我们需要重写GifDrawable，里面保存了一个该GifDrawable需要刷新的TextView的ConcurrentHashMap以及一个对该TextView列表进行刷新的Callback接口。
```java
public class EmotionGifDrawable extends GifDrawable {
    
    //setCallback保存的是弱引用，因此这里需要保存GifDrawableCallBack的强引用，使其生命周期跟GifDrawable一样
    private GifDrawableCallBack callBack;  
    
    private ConcurrentHashMap<Integer, WeakReference<TextView>> textViewMap;   //GifDrawbale需要刷新的TextView列表
    
    public EmotionGifDrawable(@NonNull Resources resources, @DrawableRes @RawRes int id) throws Resources.NotFoundException, IOException{
        super(resources, id);
        textViewMap = new ConcurrentHashMap<>();
        callBack = new GifDrawableCallBack();
        setCallback(callBack);    //设置回调函数
    }
    
    //添加到TextView刷新列表
    private void addTextView(TextView textview) {

        //利用TextView的hashCode()保存到textViewMap

    }
    
    //从TextView刷新列表中移除, ListView复用需要进行移除
    private void removeTextView(TextView textView) {
        
    }
    
    
    class GifDrawableCallBack implements Callback {

        @Override
        public void invalidateDrawable(Drawable who) {
            for(Map.Entry<Integer, WeakReference<TextView>> entry : textViewMap.entrySet()) {
                if(entry.getValue().get() != null) {
                    entry.getValue().get().invalidate();    //回调进行TextView强制刷新操作
                }
            }
        }

        @Override
        public void scheduleDrawable(Drawable who, Runnable what, long when) {

        }

        @Override
        public void unscheduleDrawable(Drawable who, Runnable what) {

        }
    }
    
}
```

对于TextView中频率最高的表情的计算可以在表情匹配的时候进行，下面只贴出关键代码

```java
	int minFrameinterval = Integer.MAX_VALUE;
	while(matcher.find()) {   //匹配到表情
		GifDrawable gifDrawable = EmotionManager.getEmotion(resources, matcher.group());   //获取表情
		if(gifDrawable != null) {
			int tempFrameInteval = gifDrawable.getDuration() / gifDrawable.getFrameByteCount();   //计算帧间隔
			if(tempFrameInteval < minFrameinterval) {
				maxFrequencyDrawable = gifDrawable;   //找到频率最高即帧间隔最短的Drawable
				minFrameinterval = tempFrameInteval;
			}
		}
	}

```


