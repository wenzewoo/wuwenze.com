+++
title = "Postman高级技巧::Pre-Request-Script &Tests-Script"
date = "2018-11-13 08:06:00"
url = "archives/269"
tags = ["Postman"]
categories = ["测试"]
+++

身为一个接口自动化测试工具，具备在运行中的动态行为不足为奇，Postman集成了一个强大的，基于NodeJS的Script引擎，利用它可以为请求以及响应添加一些动态的行为：

1）在发送请求之前，编写Pre-Request-Script，为请求参数进行加密处理、参数化等。

2）接收到请求响应后，编写Tests-Script，制定响应断言、处理返回的数据等。

大致的流程如下图：

![5bbf6ed2-ecde-4117-be9f-079209ce1091.png][]

### 实战 ###

> 现有两个接口，分别为获取Token和获取用户信息，获取Token接口参数需要计算Sign签名，该接口的返回值将成为获取用户信息接口的参数。

#### 环境变量 ####

> 为了方便的在测试环境以及开发环境中无缝切换，将相关的信息配置为两套环境变量

![2019-08-19-73729][]![2019-08-19-073729][]

#### OpenAccountApi.getUserToken ####

| 接口地址 | http://{{host}}/api2/OpenAccountApi.getUserToken    |
|------|-----------------------------------------------------|
| 请求方式 | POST                                                |
| 请求参数 | ?_app_key=[]&_time=[]&_sign=[]                      |
| 请求正文 | {"account":"账号信息"}                                  |
| 响应正文 | {"result": { "token": "","user_id": ?},"status": 0} |


按照接口约定，配置相关的Postman请求：

![f596cfda-ae53-4a52-bd50-f635b7446e17.png][]![2566bac4-2afc-4ab8-bd35-0b3674aa19bd.png][]

上图上中的`{{_time}}`、`{{_sign}}`变量目前还取不到值，因为在环境变量中还没有相关定义，

现在开始编写相关的Pre-Request-Script脚本：

![808c8606-8b41-4f38-ad06-b90311fd3068.png][]

```js
// 前置处理器：计算请求签名
var _app_secret = pm.environment.get("provider_app_secret");
var _time = (new Date()).valueOf();
var _pre_sign = 'requestBody=' + pm.request.body.raw + ',time=' + _time + ',appSecret=' + _app_secret;
var _sign = CryptoJS.MD5(_pre_sign).toString();
pm.environment.set("_time", _time);
pm.environment.set("_sign", _sign);

console.log('[Pre]OpenAccountApi.getUserToken _pre_sign='+_pre_sign+',_sign=' + _sign);
```

该脚本完成后，之前参数中使用`{{_sign}}`的就能动态的获取到值了。

请求发送成功后，需要提取响应正文的token作为下一个接口的参数，取出来放入Postman环境变量中即可。

编写相关的Tests-Script脚本：

![24ecb256-fdca-45fa-8682-240ea0c430a6.png][]

```js
// 响应断言
pm.test("Body matches token", function () {
    pm.expect(pm.response.text()).to.include(""token":");

    // 提取Token
    var result = pm.response.json().result;
    pm.environment.set("_userid", result.user_id);
    pm.environment.set("_token", result.token);
    console.log('[Tests]OpenAccountApi.getUserToken _token=' + result.token + ',user_id=' + result.user_id);
});
```

#### OpenUserApi.findById ####

| 接口地址 | http://{{host}}/api2/OpenAccountApi.getUserToken |
|------|--------------------------------------------------|
| 请求方式 | POST                                             |
| 请求参数 | ?_token=[]                                       |
| 请求正文 | {"id":用户Id}                                      |
| 响应正文 | {"result": {用户信息},"status": 0}                   |


![55e7407d-02da-443a-848f-b8f52a44f253.png][]![b36a5b9b-edd2-4196-91e0-d17dbca3ff19.png][]

第二个接口没有什么特殊处理，写了个简单的响应断言，若响应正文中包含"status":0，则表明请求成功了。

```js
console.log('[Tests]OpenUserApi.findById, requestBody=' + pm.request.body.raw);

pm.test("Body matches status", function () {
    pm.expect(pm.response.json().status).to.eql(0);
});
```

#### 最终的测试效果 ####

> 按`Ctrl + Shift + I` 可以弹出开发者面板，查看到打印的相关日志。

![82dae729-f424-48cf-90e6-98f8f4004210.png][]![07b9bd48-726c-432d-a46e-2323537de1d7.png][]

#### Collection Runner ####

![deb5d2dc-02e0-4abb-9a99-8022efd8e65b.png][]![de85e2de-a6d6-4a63-8c1f-2d2db5ce380e.png][]

#### 运行日志 ####

![7c147c64-854f-408c-ba92-cb7e3f7f9fdd.png][]


[5bbf6ed2-ecde-4117-be9f-079209ce1091.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181113/5bbf6ed2-ecde-4117-be9f-079209ce1091.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[2019-08-19-73729]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181113/f3ebf5a6-9854-46d5-b4cc-f318d081fdb8.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[2019-08-19-073729]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181113/a5fc3802-0dcd-4b42-a2ed-304a7d2fc678.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[f596cfda-ae53-4a52-bd50-f635b7446e17.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181113/f596cfda-ae53-4a52-bd50-f635b7446e17.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[2566bac4-2afc-4ab8-bd35-0b3674aa19bd.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181113/2566bac4-2afc-4ab8-bd35-0b3674aa19bd.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[808c8606-8b41-4f38-ad06-b90311fd3068.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181113/808c8606-8b41-4f38-ad06-b90311fd3068.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[24ecb256-fdca-45fa-8682-240ea0c430a6.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181113/24ecb256-fdca-45fa-8682-240ea0c430a6.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[55e7407d-02da-443a-848f-b8f52a44f253.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181113/55e7407d-02da-443a-848f-b8f52a44f253.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[b36a5b9b-edd2-4196-91e0-d17dbca3ff19.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181113/b36a5b9b-edd2-4196-91e0-d17dbca3ff19.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[82dae729-f424-48cf-90e6-98f8f4004210.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181113/82dae729-f424-48cf-90e6-98f8f4004210.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[07b9bd48-726c-432d-a46e-2323537de1d7.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181113/07b9bd48-726c-432d-a46e-2323537de1d7.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[deb5d2dc-02e0-4abb-9a99-8022efd8e65b.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181113/deb5d2dc-02e0-4abb-9a99-8022efd8e65b.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[de85e2de-a6d6-4a63-8c1f-2d2db5ce380e.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181113/de85e2de-a6d6-4a63-8c1f-2d2db5ce380e.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[7c147c64-854f-408c-ba92-cb7e3f7f9fdd.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181113/7c147c64-854f-408c-ba92-cb7e3f7f9fdd.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg