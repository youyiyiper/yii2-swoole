WebSocket Server
=======================

通过swoole与本项目可很方便的搭建websocket服务。

### 特点

* 采用Yii控制器方式编写业务代码。

### 开始使用

#### 启动脚本
```
use \tsingsun\swoole\server\Server;

defined('WEBROOT') or define('WEBROOT', __DIR__);

require(__DIR__ . '/../../vendor/autoload.php');
$config = [
    'class'=>'tsingsun\swoole\server\WebSocketServer',
    'serverType'=>'websocket',
    'port'=>9502,
    'setting' => [
        'daemonize'=>0,
        'worker_num'=>1,
    ],
];

Server::run($config,function (Server $server){
    $starter = new \tsingsun\swoole\bootstrap\WebSocketApp($server);
    //初始化函数独立,为了在启动时,不会加载Yii相关的文件,在库更新时采用reload平滑启动服务器
    $starter->init = function ($bootstrap) {
        require(__DIR__ . '/../../src/Yii.php');

        $config = yii\helpers\ArrayHelper::merge(
            require(__DIR__ . '/../config/main.php'),
            require(__DIR__ . '/../config/main-local.php')
        );
        Yii::setAlias('@webroot', WEBROOT);
        Yii::setAlias('@web', '/');
        $bootstrap->appConfig = $config;
    };
    $starter->formatData = function ($data) {

        if($data instanceof \yii\web\ForbiddenHttpException){
            return ['errors' => [['code' => $data->getCode(), 'message' => $data->getMessage()]]];
        } elseif($data instanceof \Throwable){
            return ['errors' => [['code' => $data->getCode(), 'message' => $data->getMessage()]]];
        }
            return json_encode($data);
    };
    $server->bootstrap = $starter;
//    $server->getSwoole()->
    $server->start();
});
```

* WebSocketApp

属性说明：

    - formatData 回调属性,可直接在启动脚本中进行设置,自定义接口返回数据协议.
    当未设置时,框架默认以json数据进行返回.
    
    ```php
    /**
    * 方法原型 formatData($data)
    * @param mixed|Exceptino|Throwable $data 控制器返回数据,当存在异常时为异常实例
    * @return string 通过该方法格式化的数据都将返回给调用端
    */
    formatData($data)
    ```
* 开始开发
  - 连接控制器,提供客户端的连接管理.
    ```php
    class ConnectionController extends Controller
    // 握手方法
    actionOpen()
    // 关闭连接
    actionClose()
    // 通信方法,非必要
    actionMessage()
    ```
  - 建立连接open(),对应到指定控制器的actionOpen()方法.如果请求失败,则返回错误信息,并关闭连接.
  在连接建立后,将存储本次连接客户端连接时采用的连接控制器.
  
  - 通信message(),通信时通过约定协议可自由访问其他业务控制器,当采用如下协议时:
    ```json
    {
        "route":"module/controller/action",//采用MVC方式进行路由
        "content":{} //请求数据,可为数组或对象
    }
    ```
    将指向协议中的路由.当检测非该协议时,将指向连接控制器的actionMessage(),如果未设置通信方法,将报错
  - 连接关闭,将调用连接控制器的actionClose()方法进行关闭操作.