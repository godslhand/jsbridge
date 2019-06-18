# jsbridge

##### 目标：

1. 原生为主，H5辅助快速开发

1. H5调用原生特性

1. 热更新 升级
 
1. 插件化加载

	 1. 插件功能化，已natvie特性为主，不携带业务

	 1. 固话的，整套逻辑native已有实现的，作为业务插件调用的也可开发特定的插件接口

##### 解决方案：

 借鉴cordova，将H5页面放在app的资源目录，形成一个本地的静态应用，ajax请求网络接口，并通过jsBridge 调用native特效，同时能够通过【app的动态更新资源】功能实现热更新

##### 原理:
yl
<pre>
       ios: 

           h5 调用native

                1.通过ajax或iframe形式发起类似协议的请求 'jsbridge://doAction?title=xxxxx;

                   webview拦截请求、解析出参数，并转换到相印的程序中

           native 调用js 

                // Swift
                1.webview.stringByEvaluatingJavaScriptFromString("Math.random()") 
                  // OC
                2.[webView stringByEvaluatingJavaScriptFromString:@"Math.random();"];

     android ：

             native 调用js ： webview.loadURL(“javascript:xxxxxx”)

             js 调用 native ：

                     1.通过schema方式，使用shouldOverrideUrlLoading方法对url协议进行解析。这种js的调用方式与ios的一样，使用iframe来调用native代码。 

                     2.通过在webview页面里直接注入原生js代码方式，使用@JavascriptInterface注解方法来实现。  

                     3.使用prompt,console.log,alert方式，这三个方法对js里是属性原生的，在android webview这一层是可以重写这三个方法的。一般我们使用prompt，因为这个在js里使用的不多，用来和native通讯副作用比较少。
		     
</pre>

##### 具体接口：

1. 页面调用 native 
	1. 协定协议:xxx://class/port/method?params;（android建议用第三钟，覆盖prompt）
        1. class 对应的类名
        1. port每次请求的唯一标识，回调时用来定位自己的callback函数
        1. method对应具体的方法
        1. params json格式的参数，传递到方法里面去
1. app调用 页面
    1. javascript:YLNative.onComplete(port,params);
        1. port对应前面传回来的port，唯一标识
        1. params 返回给js的参数
        1. params 结果数据返回格式:
        ```
        var resultData = {
        status: {
            code: 0,//1成功，0失败
            msg: '请求超时'//失败时候的提示，成功可为空
            },
        data: {}//数据,无数据可以为空
        };

        ```
1. H5 调用 和回调
    1. 页面调用如 ：YLNative.base.getAppName({参数可传可不传},function(data,status)
    1. 页面回调会将上面的params参数:拆分成类似ajax请求的一个data，一个stauts
    1. YLNative.js(主js)、YLNative_plugins.js（插件配置js）
    1. 插件配置如下 :
    ```
    YLNative.define('YLNative/plugin_list', function(require, exports, module) {
    module.exports = [
    
        {
            "id": "com.xxx.baseAction",
            "file": "plugins/BaseAction.js",
            "clobbers": ["baseAction"]
        },{
                  "id": "com.xxx.demo",
                  "file": "plugins/Demo.js",
                  "clobbers": ["demo"]
              }
    ];
    });
    
    ```
        1. id： 标识唯一标识，必须和插件的define 标志相对应，不能重复
        1. file：插件js的目录
        1. clobbers：命名空间，会自动依附到YLNative的空间下面，虽然写了数组但目前仅支持一个
    1. 插件JS 写法:
        ```
        cordova.define("com.xxx.baseAction", function(require, exports, module) {
         var exec = require('cordova/exec');
        module.exports = {
            takePicture:function (gallery,callback) {
                console.info("takePhoto gallery:"+gallery);
                exec(function () {
                    callback(arguments[0])
                    console.info(arguments);
                }, function () {
                    callback(arguments[0])
                }, "Camera", "takePicture", [gallery]);
             
            }
            
        };
        
        });
        
        ```
 
还需要实现的:

  静态资源的热更新功能
  更多的native接口实现

可能存在的问题:

ajax请求跨域: app设置白名单，或者直接调用app的原生像服务器发请求
android api>16(4.1) webView.getSettings().setAllowUniversalAccessFromFileURLs(true);
app 和静态应用的 会话同步
