## 使用 addJavaScriptInterface() 方法在 WebView 中绑定 Java 对象##

### 1. 说明
　　WebView 允许开发者拓展 JavaScript API 命名空间，通过 Java 定义各自的组件然后令其在 JavaScript 环境中可用。在你想接入一个平台却在 JavaScript 的方法中不可用时或想要在 JavaScript 中使用用 Java 编写的组件时，此项技术 WebView 的这个方法可以达到你的目的，例子如下：

```java
    JavaScriptInterface JavaScriptInterface = new JavaScriptInterface(this);
    myWebView = new MyWebView(this);
    myWebView.addJavaScriptInterface(JavaScriptInterface, "HybridNote");
```

　　在这个例子中，JavaScriptInterface 被绑定到 WebView 的 JavaScript 环境并且可通过 HybridNote 对象 (aka namespace) 接入。这个绑定对象的所有共有或一些特定方法在 JavaScript 代码中可接入引用。一旦这个对象被使用前面指定的函数添加到 WebView 中，这个对象将只有在 WebView 中的页面加载完成或存在已被加载完的页面时，方可在 JavaScript 中引用。

　　虽然 addJavaScriptInterface() 在设计 HybridApp 时是一个强大的方法，但是使用这个方法存在一个较大的安全隐患。这是因为同源规则 (SOP) 不适用与该方法，加上第三方 JavaScript 库或来自一个陌生域名的 iframe 可能在 Java 层访问这些被暴露的方法。因此，攻击者可通过一个 XSS 漏洞执行原生代码或者注入病毒代码到应用程序中。

　　JavaScript 层中暴露的 Java 对象的所有公有方法在 Android 版本低于 JerryBean MRI(API Level 17) 以下时可访问。而在 Google API 17 （4.２）以上，暴露的函数必须通过 @JavaScriptInterface 注释来防止方法的暴露。

### 2. 使用
　　下面通过使用一个实例来介绍如何在webView调用java对象。下面这一个例子是介绍如何通过获取WebView中页面的标题作为参数然后调用Java对象将标题显示在TextView中，虽然通过重写WebChromeClient的onReceivedTitle可以获取页面的标题并显示，但是该方法中在页面回退（goBack）的时候并不一定会被调用，回退的时候显示的还是上一个页面的标题，因此该方法也有一定的局限性，即使可以通过一个栈记录访问过的URL和页面标题。
	
 　使用 addJavaScriptInterface() 方法则可以解决这一个问题，因为页面标题都保存在页面内容中，因此可以通过将标题作为参数去调用Java对象的某一个方法，而该方法的功能则是显示标题，具体代码如下。


```java
public class MainActivity extends AppCompatActivity {
    
    private WebView webview;
    private TextView tvTitle;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        webview = (WebView)findViewById(R.id.webView);
        tvTitle = (TextView)findViewById(R.id.tvTitle);
        
        WebSettings settings = webview.getSettings();
        settings.setJavaScriptEnabled(true);// 这样网页就可加载JavaScript了
        settings.setJavaScriptCanOpenWindowsAutomatically(true);
        settings.setBuiltInZoomControls(true);// 显示放大缩小按钮
        settings.setSupportZoom(true);// 允许放大缩小
        settings.setSupportMultipleWindows(true);

        webview.addJavascriptInterface(new GetTitle(), "getTitle"); // 向webview注册一个Java对象
        webview.setWebViewClient(new WebViewClient(){
            @Override
            public boolean shouldOverrideUrlLoading(WebView view, String url) {
                view.loadUrl(url);
                return true;
            }

            @Override
            public void onPageFinished(WebView view, String url) {
				//注入一段JavaScript，该代码主要是调用Java对象的一个方法并将页面标题作为参数
                view.loadUrl("javascript:window.getTitle.onGetTitle("
                        + "document.getElementsByTagName('title')[0].innerHTML" + ");");
                super.onPageFinished(view, url);
            }
        });

        webview.setWebChromeClient(new WebChromeClient(){
            @Override
            public void onReceivedTitle(WebView view, String title) {
                tvTitle.setText(title);
            }
        });

        webview.loadUrl("https://www.baidu.com/");
    }


    @Override
    public boolean onKeyDown(int keyCode, KeyEvent event) {
        if(keyCode == KeyEvent.KEYCODE_BACK) {
            if(webview.canGoBack()) {
                webview.goBack();
            } else {
                finish();
            }
            return true;

        }
        return super.onKeyDown(keyCode, event);
    }

    //注入JavaScript的Java类
    class GetTitle {
        @JavascriptInterface
        public void onGetTitle(String title) {
           tvTitle.setText(title);     //设置标题
        }
    }

    @Override
    protected void onDestroy() {
        webview.destroy();
        super.onDestroy();

    }
}

```
