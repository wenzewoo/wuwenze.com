+++
title = "DingTalk::通讯录单向同步实现示例"
date = "2018-07-25 14:32:00"
url = "archives/21"
tags = ["DingTalk"]
categories = ["后端"]
+++

最近项目中需要实现对接钉钉，并实现单向通讯录同步（`钉钉服务器` \-> `对接平台`）本文通过一个简单的案例快速实现相关的DEMO (本文主要实现与钉钉对接)。

> 钉钉API：[https://open-doc.dingtalk.com/docs/doc.htm?spm=a219a.7629140.0.0.LucpAu&treeId=385&articleId=104975&docType=1\#s7][https_open-doc.dingtalk.com_docs_doc.htm_spm_a219a.7629140.0.0.LucpAu_treeId_385_articleId_104975_docType_1_s7]

### 流程示意图 ###

![56eb13a3-5c17-4507-97fe-051a19194f70.png][]

### 准备工作 ###

在使用回调接口前，需要做以下准备工作：

 *  提供一个接收消息的RESTful接口。
 *  调用钉钉API，主动注册回调通知。
 *  因为涉及到消息的加密解密，默认的JDK存在一些限制，先要替换JCE无限制权限策略文件。
 *  内网穿透映射本地RESTful接口到公网，推荐使用`Ngrok`: [http://ngrok.ciqiuwl.cn/][http_ngrok.ciqiuwl.cn]

> 附：JCE无限制权限策略文件替换返回
> 
> JDK6的下载地址：[http://www.oracle.com/technetwork/java/javase/downloads/jce-6-download-429243.html][http_www.oracle.com_technetwork_java_javase_downloads_jce-6-download-429243.html]  
> JDK7的下载地址：[http://www.oracle.com/technetwork/java/javase/downloads/jce-7-download-432124.html][http_www.oracle.com_technetwork_java_javase_downloads_jce-7-download-432124.html]  
> JDK8的下载地址：[http://www.oracle.com/technetwork/java/javase/downloads/jce8-download-2133166.html][http_www.oracle.com_technetwork_java_javase_downloads_jce8-download-2133166.html]  
> 下载后解压，可以看到`local_policy.jar`和`US_export_policy.jar`以及readme.txt。  
> 如果安装的是JRE，将两个jar文件放到`%JRE_HOME%/lib/security`目录下覆盖原来的文件，  
> 如果安装的是JDK，将两个jar文件放到`%JDK_HOME%/jre/lib/security`目录下覆盖原来文件。

### 具体实现 ###

#### 提供回调接口 ####

```java
/**
 * @author wwz
 * @version 1 (2018/7/26)
 * @since Java7
 */
@Slf4j
@RestController
public class DingTalkCallbackRest {

    @PostMapping("/dingtalk/receive")
    public Map<String, ? extends Serializable> receive(//
          String signature, String timestamp, String nonce,@RequestBody String requestBody) {
        Assert.notNull(signature, "signature is null.");
        Assert.notNull(timestamp, "timestamp is null.");
        Assert.notNull(nonce, "nonce is null.");
        Assert.notNull(requestBody, "requestBody is null.");

        log.info("#receive 接收密文：{}", requestBody);
        DingTalkEncryptor dingTalkEncryptor = new DingTalkEncryptor(//
                DingTalkConst.CALLBACK_TOKEN, DingTalkConst.CALLBACK_AES_KEY, DingTalkConst.CORP_ID);
        JSONObject jsonEncrypt = JSON.parseObject(requestBody);
        String encryptMessage = dingTalkEncryptor.getDecryptMsg(signature, timestamp, nonce, jsonEncrypt.getString("encrypt"));
        log.info("#receive 密文解密后：{}", encryptMessage);

        // TODO: 异步处理报文，解析相关信息

        // 返回加密后的success (快速响应)
        return dingTalkEncryptor.getEncryptedMsg("success", Long.parseLong(timestamp), nonce);
    }
}
```

接口写好之后，还需要将接口暴露在公网，如此钉钉服务器才能进行调用，下为内网穿透示意图：

![8ea2177a-510b-4d55-a6a0-55474ea18e96.png][]

钉钉为我们开发者提供了一个Ngrok服务，在`https://github.com/open-dingtalk/pierced.git`，按照操作文章指引配置即可。

我在这边使用的是其他的Ngrok服务，官网地址是`http://ngrok.ciqiuwl.cn/`，配置后启动如下图所示:

![90e53afd-8787-42c7-90fc-16fd75b28d30.png][]

将本地的http://127.0.0.1:8080映射到http://wuwz.ngrok.xiaomiqiu.cn，最终提供给钉钉的回调接口地址即为：http://wuwz.ngrok.xiaomiqiu.cn/dingtalk/receive

以上准备工作完后成，就可以将接口启动起来，继续后续的操作。

#### 主动注册回调接口 ####

> 写一个测试方法，将http://wuwz.ngrok.xiaomiqiu.cn/dingtalk/receive注册到钉钉，后续钉钉相关的消息都会推送到此处。

```java
/**
 * @author wwz
 * @version 1 (2018/7/27)
 * @since Java7
 */
public class TestRegisterCallback {

    public static void main(String[] args) {
        // 获取Token
        String accessToken = DingTalkApi.getAccessTokenCache();

        // 先删除之前注册的回调接口
        DingTalkApi.removeCallback(accessToken);

        // 注册新的回调接口
        String callbackToken = DingTalkConst.CALLBACK_TOKEN;
        String callbackAesKey = DingTalkConst.CALLBACK_AES_KEY;
        String callbackUrl = "http://wuwz.ngrok.xiaomiqiu.cn/dingtalk/receive";
        DingTalkCallbackTag[] callbacktags = {
                DingTalkCallbackTag.USER_ADD_ORG, // 增加用户
                DingTalkCallbackTag.USER_MODIFY_ORG, // 修改用户
                DingTalkCallbackTag.USER_LEAVE_ORG // 用户离职
        };
        DingTalkApi.registerCallback(accessToken, callbackToken, callbackAesKey, callbackUrl, callbackTags);
    }
}
```

执行代码，如果一切不出意外的话，就注册成功了（注册的过程中需**保证callbackUrl可以正常访问**,因为首次会向该接口发送一条`check_url事件`，验证其合法性）

```bash
// 获取Token
13:44:41.342 [main] INFO com.wuwenze.dingtalk.api.DingTalkApi - {"access_token":"9990578f789c3fb1a9d974c268df5029","errcode":0.0,"errmsg":"ok","expires_in":7200.0}
// 先删除之前注册的回调接口
13:44:41.438 [main] INFO com.wuwenze.dingtalk.api.DingTalkApi - {"errcode":0.0,"errmsg":"ok"}
// 注册新的回调接口
13:44:41.888 [main] INFO com.wuwenze.dingtalk.api.DingTalkApi - {"errcode":0.0,"errmsg":"ok"}
13:44:41.893 [main] INFO com.wuwenze.dingtalk.api.DingTalkApi - ##registerCallback 注册回调接口 -> url: http://wuwz.ngrok.xiaomiqiu.cn/dingtalk/receive, tags: tag: user_add_org, describe: 通讯录用户增加 + tag: user_modify_org, describe: 通讯录用户更改 + tag: user_leave_org, describe: 通讯录用户离职
```

另外再来观察一下回调接口是否收到checkUrL消息：

```bash
2018-07-27 13:44:41.823  INFO 2392 --- [nio-8080-exec-1] c.w.dingtalk.rest.DingTalkCallbackRest   : ##receive 接收密文：{"encrypt":"JfRo/wn+E1agXgk1uN5UQP/WDv0RvWnw8TgXC/ucatBxYm54OSUcGn5uTGCVMaGIN6Lv24ZOujH/uixB39AKxjXWgzdJQ1Eq4HD0EIJFG+QY8mjcCltvhX0QfhisFlll"}
2018-07-27 13:44:41.823  INFO 2392 --- [nio-8080-exec-1] c.w.dingtalk.rest.DingTalkCallbackRest   : ##receive 密文解密后：{"EventType":"check_url"}
```

#### 测试注册的通讯录事件 ####

> 在上一步中，注册了`USER_ADD_ORG` (增加用户)、`USER_MODIFY_ORG` (修改用户)、`USER_LEAVE_ORG` (用户离职|删除)三个事件

打开钉钉后台管理，在通讯录中新增一个用户：

![e50a3ee8-f173-4624-89bc-7b4c2de91435.png][]

保存成功后，在回调接口中则马上收到了该事件的通知消息：

```bash
2018-07-27 13:49:55.985  INFO 2392 --- [nio-8080-exec-3] c.w.dingtalk.rest.DingTalkCallbackRest   : ##receive 接收密文：{"encrypt":"g6RsagVKTVUS2Gg7B1JSn81uJPgCpPKoaRN4kps4cMpp6CuqW1QahaDP8TcnwDP2fYyG0gwLFvF5cOWbn+lKX2kq4UYe5m08BB/FWw8lALV/4LYu7RI6OARCFDTsllBTs4W6/OUv+9AyYlWGmwK2ZYnXoFyiK4DqFt6jenp45NCXwvSgssjn8RsD/3E7kfw5DL/mfr4L3hkaBysmkU2ohaFFEqBO1r63cj+mONLsD8Dvr2lAsefBoMdZ2JV5sIIePuKhz08G6KnJDvkAqcm59naV6AIbDLouWrBK7upCP7Q="}
2018-07-27 13:49:55.985  INFO 2392 --- [nio-8080-exec-3] c.w.dingtalk.rest.DingTalkCallbackRest   : ##receive 密文解密后：{"TimeStamp":"1532670599144","CorpId":"dingb9875d6606f892ed35c2f4657eb6378f","UserId":["202844352662984130"],"EventType":"user_add_org"}
```

#### 后续同步逻辑 ####

在上面的例子中新增用户后，收到的报文解密后的信息为只包含事件类型和用户ID，所以后面还需要主动调用钉钉获取用户详情的接口，再做具体的同步逻辑，这里就不再往下写了，贴一下相关的API接口吧：

[https://open-doc.dingtalk.com/docs/doc.htm?spm=a219a.7629140.0.0.Y7lQU8&treeId=385&articleId=106816&docType=1][https_open-doc.dingtalk.com_docs_doc.htm_spm_a219a.7629140.0.0.Y7lQU8_treeId_385_articleId_106816_docType_1]

![2019-08-19-073459][]

### 相关API工具封装 ###

> 下面罗列了以上示例中用到的工具类封装，不再具体讲解，直接贴代码

#### DingTalkConst ####

> 常量池

```java
public class DingTalkConst {
    public final static String CORP_ID = "dingb9875d6606f892ed35c2f4657eb6378f";
    public final static Object CORP_SECRET = "到钉钉查看";
    public final static String CALLBACK_TOKEN = "token"; // 回调Token
    public final static String CALLBACK_AES_KEY = "xxxxx7p5qnb6zs3xxxxxlkfmxqfkv23d40yd0xxxxxx"; // 回调秘钥，43个随机字符
}
```

#### DingTalkCallbackTag ####

> 可供注册的回调事件类型枚举

```java
public enum DingTalkCallbackTag {
    USER_ADD_ORG("通讯录用户增加"),
    USER_MODIFY_ORG("通讯录用户更改"),
    USER_LEAVE_ORG("通讯录用户离职"),
    ORG_ADMIN_ADD("通讯录用户被设为管理员"),
    ORG_ADMIN_REMOVE("通讯录用户被取消设置管理员"),
    ORG_DEPT_CREATE("通讯录企业部门创建"),
    ORG_DEPT_MODIFY("通讯录企业部门修改"),
    ORG_DEPT_REMOVE("通讯录企业部门删除"),
    ORG_REMOVE("企业被解散"),
    ORG_CHANGE("企业信息发生变更"),
    LABEL_USER_CHANGE("员工角色信息发生变更"),
    LABEL_CONF_ADD("增加角色或者角色组"),
    LABEL_CONF_DEL("删除角色或者角色组"),
    LABEL_CONF_MODIFY("修改角色或者角色组");

    private String describe;
    DingTalkCallbackTag(String describe) {
        this.describe = describe;
    }

    public String getDescribe() {
        return describe;
    }

    public void setDescribe(String describe) {
        this.describe = describe;
    }

    @Override
    public String toString() {
        return super.toString().toLowerCase();
    }

    public String toInfoString() {
        return String.format("tag: %s, describe: %s", this.toString(), this.getDescribe());
    }
}
```

#### DingTalkEncryptor ####

> 钉钉消息加密解密工作类

```java
/**
 * @author wwz
 * @version 1 (2018/7/26)
 * @since Java7
 */
public class DingTalkEncryptor {
    private static final Charset CHARSET = Charset.forName("UTF-8");
    private byte[] aesKey;
    private String token;
    private String corpId;

    /**
     * ask getPaddingBytes key固定长度
     **/
    private static final Integer AES_ENCODE_KEY_LENGTH = 43;
    /**
     * 加密随机字符串字节长度
     **/
    private static final Integer RANDOM_LENGTH = 16;

    /**
     * 构造函数
     *
     * @param token          钉钉开放平台上，开发者设置的token
     * @param encodingAesKey 钉钉开放台上，开发者设置的EncodingAESKey
     * @param corpId         ISV进行配置的时候应该传对应套件的SUITE_KEY(第一次创建时传的是默认的CREATE_SUITE_KEY)，普通企业是Corpid
     */
    public DingTalkEncryptor(String token, String encodingAesKey, String corpId) {
        if (null == encodingAesKey || encodingAesKey.length() != AES_ENCODE_KEY_LENGTH) {
            throw new IllegalArgumentException("encodingAesKey is null");
        }
        this.token = token;
        this.corpId = corpId;
        this.aesKey = Base64.decode(encodingAesKey + "=");
    }

    /**
     * 将和钉钉开放平台同步的消息体加密,返回加密Map
     *
     * @param message 传递的消息体明文
     * @param timeStamp 时间戳
     * @param nonce     随机字符串
     * @return
     */
    public Map<String, ? extends Serializable> getEncryptedMsg(String message, Long timeStamp, String nonce) {
        Assert.notNull(message, "plaintext is null");
        Assert.notNull(timeStamp, "timeStamp is null");
        Assert.notNull(nonce, "nonce is null");
        String encrypt = encrypt(getRandomStr(RANDOM_LENGTH), message);
        String signature = getSignature(token, String.valueOf(timeStamp), nonce, encrypt);
        return ImmutableMap.of(
                "msg_signature", signature, //
                "encrypt", encrypt, //
                "timeStamp", timeStamp,//
                "nonce", nonce);
    }

    /**
     * 密文解密
     *
     * @param msgSignature 签名串
     * @param timeStamp    时间戳
     * @param nonce        随机串
     * @param encryptMsg   密文
     * @return 解密后的原文
     */
    public String getDecryptMsg(String msgSignature, String timeStamp, String nonce, String encryptMsg) {
        // 校验签名
        String signature = getSignature(token, timeStamp, nonce, encryptMsg);
        if (!signature.equals(msgSignature)) {
            throw new RuntimeException("校验签名失败。");
        }
        // 解密
        return decrypt(encryptMsg);
    }

    private String encrypt(String random, String plaintext) {
        try {
            byte[] randomBytes = random.getBytes(CHARSET);
            byte[] plainTextBytes = plaintext.getBytes(CHARSET);
            byte[] lengthByte = int2Bytes(plainTextBytes.length);
            byte[] corpidBytes = corpId.getBytes(CHARSET);
            ByteArrayOutputStream byteStream = new ByteArrayOutputStream();
            byteStream.write(randomBytes);
            byteStream.write(lengthByte);
            byteStream.write(plainTextBytes);
            byteStream.write(corpidBytes);
            byte[] padBytes = PKCS7Padding.getPaddingBytes(byteStream.size());
            byteStream.write(padBytes);
            byte[] unencrypted = byteStream.toByteArray();
            byteStream.close();
            Cipher cipher = Cipher.getInstance("AES/CBC/NoPadding");
            SecretKeySpec keySpec = new SecretKeySpec(aesKey, "AES");
            IvParameterSpec iv = new IvParameterSpec(aesKey, 0, 16);
            cipher.init(Cipher.ENCRYPT_MODE, keySpec, iv);
            byte[] encrypted = cipher.doFinal(unencrypted);
            return Base64.encode(encrypted);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    /**
     * 对密文进行解密.
     * @param text 需要解密的密文
     * @return 解密得到的明文
     */
    private String decrypt(String text) {
        byte[] originalArr;
        try {
            // 设置解密模式为AES的CBC模式
            Cipher cipher = Cipher.getInstance("AES/CBC/NoPadding");
            SecretKeySpec keySpec = new SecretKeySpec(aesKey, "AES");
            IvParameterSpec iv = new IvParameterSpec(Arrays.copyOfRange(aesKey, 0, 16));
            cipher.init(Cipher.DECRYPT_MODE, keySpec, iv);
            // 使用BASE64对密文进行解码, 解密
            originalArr = cipher.doFinal(Base64.decode(text));
        } catch (Exception e) {
            throw new RuntimeException("计算解密文本错误");
        }

        String plainText;
        String fromCorpid;
        try {
            // 去除补位字符
            byte[] bytes = PKCS7Padding.removePaddingBytes(originalArr);
            // 分离16位随机字符串,网络字节序和corpId
            byte[] networkOrder = Arrays.copyOfRange(bytes, 16, 20);
            int plainTextLegth = bytes2int(networkOrder);
            plainText = new String(Arrays.copyOfRange(bytes, 20, 20 + plainTextLegth), CHARSET);
            fromCorpid = new String(Arrays.copyOfRange(bytes, 20 + plainTextLegth, bytes.length), CHARSET);
        } catch (Exception e) {
            throw new RuntimeException("计算解密文本长度错误");
        }

        // corpid不相同的情况
        if (!fromCorpid.equals(corpId)) {
            throw new RuntimeException("计算文本密码错误");
        }
        return plainText;
    }

    /**
     * 数字签名
     * @param token     isv token
     * @param timestamp 时间戳
     * @param nonce     随机串
     * @param encrypt   加密文本
     * @return
     */
    public String getSignature(String token, String timestamp, String nonce, String encrypt) {
        try {
            String[] array = new String[]{token, timestamp, nonce, encrypt};
            Arrays.sort(array);
            StringBuffer sb = new StringBuffer();
            for (int i = 0; i < 4; i++) {
                sb.append(array[i]);
            }
            String str = sb.toString();
            MessageDigest md = MessageDigest.getInstance("SHA-1");
            md.update(str.getBytes());
            byte[] digest = md.digest();

            StringBuffer hexstr = new StringBuffer();
            String shaHex = "";
            for (int i = 0; i < digest.length; i++) {
                shaHex = Integer.toHexString(digest[i] & 0xFF);
                if (shaHex.length() < 2) {
                    hexstr.append(0);
                }
                hexstr.append(shaHex);
            }
            return hexstr.toString();
        } catch (Exception e) {
           throw new RuntimeException(e);
        }
    }

    public static String getRandomStr(int count) {
        String base = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789";
        Random random = new Random();
        StringBuffer sb = new StringBuffer();
        for (int i = 0; i < count; i++) {
            int number = random.nextInt(base.length());
            sb.append(base.charAt(number));
        }
        return sb.toString();
    }


    /**
     * int转byte数组,高位在前
     */
    public static byte[] int2Bytes(int count) {
        byte[] byteArr = new byte[4];
        byteArr[3] = (byte) (count & 0xFF);
        byteArr[2] = (byte) (count >> 8 & 0xFF);
        byteArr[1] = (byte) (count >> 16 & 0xFF);
        byteArr[0] = (byte) (count >> 24 & 0xFF);
        return byteArr;
    }

    /**
     * 高位在前bytes数组转int
     * @param byteArr
     * @return
     */
    public static int bytes2int(byte[] byteArr) {
        int count = 0;
        for (int i = 0; i < 4; i++) {
            count <<= 8;
            count |= byteArr[i] & 0xff;
        }
        return count;
    }
}
```

#### PKCS7Padding ####

```java
/**
 * @author wwz
 * @version 1 (2018/7/10)
 * @since Java7
 */
public class PKCS7Padding {
    private final static Charset CHARSET    = Charset.forName("utf-8");
    private final static int     BLOCK_SIZE = 32;

    /**
     * 填充mode字节
     * @param count
     * @return
     */
    public static byte[] getPaddingBytes(int count) {
        int amountToPad = BLOCK_SIZE - (count % BLOCK_SIZE);
        if (amountToPad == 0) {
            amountToPad = BLOCK_SIZE;
        }
        char padChr = chr(amountToPad);
        String tmp = new String();
        for (int index = 0; index < amountToPad; index++) {
            tmp += padChr;
        }
        return tmp.getBytes(CHARSET);
    }

    /**
     * 移除mode填充字节
     * @param decrypted
     * @return
     */
    public static byte[] removePaddingBytes(byte[] decrypted) {
        int pad = (int) decrypted[decrypted.length - 1];
        if (pad < 1 || pad > BLOCK_SIZE) {
            pad = 0;
        }
        return Arrays.copyOfRange(decrypted, 0, decrypted.length - pad);
    }

    private static char chr(int a) {
        byte target = (byte) (a & 0xFF);
        return (char) target;
    }
}
```

#### DingTalkApi ####

> 钉钉开放API简易封装 (仅供测试)

```java
/**
 * @author wwz
 * @version 1 (2018/7/26)
 * @since Java7
 */
@Slf4j
public class DingTalkApi {

    private final static LoadingCache<String, String> mTokenCache = //
            CacheBuilder.newBuilder()//
                    .maximumSize(100)//
                    .expireAfterAccess(7200, TimeUnit.SECONDS)//
                    .build(new CacheLoader<String, String>() {
                        @Override
                        public String load(String key) throws Exception {
                            // key:corpId#corpSecret
                            String[] params = key.split("#");
                            if (params.length != 2) {
                                throw new RuntimeException("#loadTokenCache error");
                            }
                            return getAccessToken(params[0], params[1]);
                        }
                    });



    public static String getAccessToken(String corpId, String corpSecret) {
        String url = String.format("https://oapi.dingtalk.com/gettoken?corpid=%s&corpsecret=%s",corpId, corpSecret);
        JSONObject jsonObject = HttpClient.get(url).asBean(JSONObject.class);
        assertDingTalkJSONObject(jsonObject);
        return jsonObject.getString("access_token");
    }

    public static String getAccessTokenCache() {
        try {
            return mTokenCache.get(DingTalkConst.CORP_ID + "#" + DingTalkConst.CORP_SECRET);
        } catch (ExecutionException e) {
            return null;
        }
    }

    public static void registerCallback(String accessToken, String callbackToken, String callbackAesKey, String url, DingTalkCallbackTag ... tags) {
        Assert.notNull(accessToken, "accessToken is null");
        Assert.notNull(callbackToken, "callbackToken is null");
        Assert.notNull(callbackAesKey, "callbackAesKey is null");
        Assert.notNull(url, "url is null");
        if (tags.length < 1) {
            throw new IllegalArgumentException("至少指定一个回调事件类型。");
        }
        String[] callbackTagArray = new String[tags.length];
        for (int i = 0; i < tags.length; i++) {
            callbackTagArray[i] = tags[i].toString();
        }
        ImmutableMap<String, Serializable> params = ImmutableMap.of(//
                "call_back_tag", callbackTagArray,//
                "token", callbackToken,//
                "aes_key", callbackAesKey, //
                "url", url//
        );
        String apiUrl =  "https://oapi.dingtalk.com/call_back/register_call_back?access_token=" + accessToken;
        assertDingTalkJSONObject(//
                HttpClient.textBody(apiUrl).json(JSON.toJSONString(params)).asBean(JSONObject.class)
        );
        log.info("#registerCallback 注册回调接口 -> url: {}, tags: {}", url, showTagsInfo(tags));
    }

    private static String showTagsInfo(DingTalkCallbackTag ... tags) {
        StringBuffer stringBuffer = new StringBuffer();
        for (DingTalkCallbackTag tag : tags) {
            stringBuffer.append(tag.toInfoString()).append(" + ");
        }
        return stringBuffer.toString();
    }

    public static void removeCallback(String accessToken) {
        String apiUrl = "https://oapi.dingtalk.com/call_back/delete_call_back?access_token=" + accessToken;
        assertDingTalkJSONObject(//
                HttpClient.get(apiUrl).asBean(JSONObject.class)
        );
    }

    public static void removeCallback() {
        removeCallback(getAccessTokenCache());
    }

    public static void registerCallback(String url, DingTalkCallbackTag ... tags) {
        registerCallback(getAccessTokenCache(), DingTalkConst.CALLBACK_TOKEN, DingTalkConst.CALLBACK_AES_KEY, url, tags);
    }

    private static void assertDingTalkJSONObject(JSONObject jsonObject) {
        log.info(jsonObject.toJSONString());
        int errcode = jsonObject.getIntValue("errcode");
        if (errcode != 0) {
            throw new RuntimeException(jsonObject.getString("errmsg"));
        }
    }
}
```


[https_open-doc.dingtalk.com_docs_doc.htm_spm_a219a.7629140.0.0.LucpAu_treeId_385_articleId_104975_docType_1_s7]: https://open-doc.dingtalk.com/docs/doc.htm?spm=a219a.7629140.0.0.LucpAu&treeId=385&articleId=104975&docType=1#s7
[56eb13a3-5c17-4507-97fe-051a19194f70.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20180725/56eb13a3-5c17-4507-97fe-051a19194f70.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[http_ngrok.ciqiuwl.cn]: http://ngrok.ciqiuwl.cn/
[http_www.oracle.com_technetwork_java_javase_downloads_jce-6-download-429243.html]: http://www.oracle.com/technetwork/java/javase/downloads/jce-6-download-429243.html
[http_www.oracle.com_technetwork_java_javase_downloads_jce-7-download-432124.html]: http://www.oracle.com/technetwork/java/javase/downloads/jce-7-download-432124.html
[http_www.oracle.com_technetwork_java_javase_downloads_jce8-download-2133166.html]: http://www.oracle.com/technetwork/java/javase/downloads/jce8-download-2133166.html
[8ea2177a-510b-4d55-a6a0-55474ea18e96.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20180725/8ea2177a-510b-4d55-a6a0-55474ea18e96.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[90e53afd-8787-42c7-90fc-16fd75b28d30.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20180725/90e53afd-8787-42c7-90fc-16fd75b28d30.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[e50a3ee8-f173-4624-89bc-7b4c2de91435.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20180725/e50a3ee8-f173-4624-89bc-7b4c2de91435.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[https_open-doc.dingtalk.com_docs_doc.htm_spm_a219a.7629140.0.0.Y7lQU8_treeId_385_articleId_106816_docType_1]: https://open-doc.dingtalk.com/docs/doc.htm?spm=a219a.7629140.0.0.Y7lQU8&treeId=385&articleId=106816&docType=1
[2019-08-19-073459]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20180725/b8b527d0-88db-4931-9432-a7c9ed71a464.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg