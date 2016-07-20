## ImageSpan图文居中对齐
ImageSpan类为TextView提供了图文混排的形式，在ImageSpan的构造函数中提供了一个参数 int verticalAlignment，表示垂直对齐方式，有两个参数 ALIGN_BASELINE、ALIGN_BOTTOM 分别为顶部、底部对齐，但是没有居中对齐的参数（其实会找到这篇文章的人应该知道这点了。。）
下面说说我的实现思路及方法

1.根据构造函数verticalAlignment参数找到影响对齐方式的代码   
```java	
    
	public ImageSpan(Context context, int resourceId, int verticalAlignment) {  
	    super(verticalAlignment);  
	    mContext = context;  
	    mResourceId = resourceId;  
	} 

```
	
    


