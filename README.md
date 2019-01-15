# ----weixinpay
微信公众号支付
> 开发公众号支付时，在统一下单接口中要求必传用户openid，而获取openid则需要在公众平台设置获取openid的域名，只有被设置过的域名才是一个有效的获取openid的域名，否则将获取失败,这也是不好测试的原因

## 后台接口
```
@RestController
@RequestMapping("/weixinGZHpay")
public class WeixinGZHpay {


    @Value("${appid}")
    private String appid;  //公众账号id

    @Value("${mchid}")
    private String mchId;  //商户号

    @Value("${weixinKey}")
    private String weixinKey;  //密匙

    @Value("${weixinAppSecret}")
    private String weixinAppSecret; //	公众号的appsecret

    @Value("https://api.mch.weixin.qq.com/pay/unifiedorder")
    private String unifiedorderUrl; //统一下单接口

    @Value("weixin.oauth2.url=https://open.weixin.qq.com/connect/oauth2/authorize")
    private String weixinOauth2Url;  //网页授权获取code

    @Value("weixin.oauth2.access_token_url=https://api.weixin.qq.com/sns/oauth2/access_token")
    private String accessTokenUrl;   //通过code获取access_token_url和openid

    @Value("https://127.0.0.1:8080/weixinGZHpay/gzhPay")
    private String redirectUrl;  //授权后重定向的回调链接地址， 请使用 urlEncode 对链接进行处理


    /**
     * //第一步：用户同意授权，获取code
     * 参考:https://mp.weixin.qq.com/wiki?t=resource/res_main&id=mp1421140842
     * 如果用户同意授权，页面将跳转至 redirect_uri/?code=CODE&state=STATE。
     *
     * @param httpServletRequest
     * @param httpServletResponse
     * @param orderId             订单id,先生成自己业务的订单
     */
    public void getCode(HttpServletRequest httpServletRequest, HttpServletResponse
            httpServletResponse, String orderId) {
        try {
            StringBuffer sb = new StringBuffer();
            sb.append(weixinOauth2Url).append("?").append
                    ("appid=").append(appid).append("&redirect_uri=").append
                    (URLEncoder.encode(redirectUrl, "UTF-8"));
            sb.append("&response_type=code&scope=snsapi_base&state=").append(orderId).append
                    ("#wechat_redirect");        //我使用静默授权
            StringBuilder sbHtml = new StringBuilder();
            sbHtml.append("<form id=\"weixinggongzhonghao\" name=\"weixinggongzhonghao\" " +
                    "action=\"" + sb.toString()
                    + "\" method=\"" + "post" + "\">");
            sbHtml.append("<input type=\"submit\" value=\"" + "payButton" + "\" " +
                    "style=\"display:none;\"></form>");
            sbHtml.append("<script>document.forms['weixinggongzhonghao'].submit();</script>");
            // 直接将完整的表单html输出到页面
            httpServletResponse.setContentType("text/html;charset=utf-8");
            httpServletResponse.getWriter().write(sbHtml.toString());
            httpServletResponse.getWriter().flush();
        } catch (Exception e) {

        }
    }

    /**
     * 第二步：通过code换取网页授权access_token和用户openid，并下单获取调起支付的参数，进入下单页调起微信支付
     *
     * @param httpServletRequest
     * @param httpServletResponse
     * @param state               从第一步获取,实际为订单id,通过订单id获取商品信息
     */
    @RequestMapping("/gzhPay")
    public String gzhPay(HttpServletRequest httpServletRequest, HttpServletResponse
            httpServletResponse, String code, String state, Writer writer, Model model)
            throws Exception {
        String mapStr = "";
        try {
            // // 换区微信access_token和用户openid
            StringBuffer sb = new StringBuffer();
            sb.append(accessTokenUrl).append("?appid=").append(appid);
            sb.append("&secret=").append(weixinAppSecret);
            sb.append("&code=").append(code);
            sb.append("&grant_type=authorization_code");
            String result = HttpUtil.doPost(accessTokenUrl, sb.toString());
            JSONObject json = JSONObject.parseObject(result);
            String openid = json.getString("openid");
            //获得openid调用微信统一下单接口
            HashMap<String, String> dataMap = new HashMap<>();
            dataMap.put("appid", appid); //公众账号ID
            dataMap.put("mch_id", mchId); //商户号
            dataMap.put("nonce_str", WXPayUtil.generateNonceStr()); //随机字符串，长度要求在32位以内。
            dataMap.put("body", "手机"); //商品描述,通过订单id获得
            dataMap.put("out_trade_no", state); //商品订单号
            dataMap.put("total_fee", "1"); //商品金,通过订单id获得
            dataMap.put("spbill_create_ip", HttpUtil.getIpAddress(httpServletRequest)); //客户端ip
            dataMap.put("notify_url", "https://127.0.0.1:8080/weixinGZHpay/notifyUrl"); //通知地址
            // (需要是外网可以访问的)
            dataMap.put("trade_type", "JSAPI"); //交易类型
            dataMap.put("openid", openid); //商户号
            //生成签名
            String signature = WXPayUtil.generateSignature(dataMap, weixinKey);
            dataMap.put("sign", signature);//签名
            //将类型为map的参数转换为xml
            String requestXml = WXPayUtil.mapToXml(dataMap);
            //发送参数,调用微信统一下单接口,返回xml
            String responseXml = HttpUtil.doPost(unifiedorderUrl, requestXml);
            System.out.print(responseXml);
            Map<String, String> map = WXPayUtil.xmlToMap(responseXml);
            if ("FAIL".equals(map.get("return_code"))) {
                mapStr = map.get("return_msg");
                writer.write(mapStr);
                return "";
            }
            if ("FAIL".equals(map.get("result_code"))) {
                mapStr = map.get("err_code_des");
                writer.write(mapStr);
                return "";
            }
            if ("".equals(map.get("prepay_id")) || map.get("prepay_id") == null) {
                writer.write("prepay_id 为空");
                return "";
            }
            //成功之后,提取prepay_id,重点就是这个
            HashMap<String, String> params = new HashMap<>();
            params.put("appId", appid);
            params.put("nonceStr", WXPayUtil.generateNonceStr());
            params.put("package", map.get("prepay_id"));
            params.put("signType", "MD5");
            params.put("timeStamp", String.valueOf(System.currentTimeMillis() / 1000));
            //重新签名
            String paySign = WXPayUtil.generateSignature(params, weixinKey);
            params.put("paySign", paySign);
            //传给前端页面
            //在微信浏览器里面打开H5网页中执行JS调起支付。接口输入输出数据格式为JSON。
            model.addAttribute("param", JSON.toJSON(params));
            return "weixinpay/weixinGZHpay";
            //JS API的返回结果get_brand_wcpay_request为ok及表示用户成功完成支付,展示支付成功页
            // 下一步就是在后台回调接口处理订单状态
        } catch (Exception e) {
            return "异常";
        }
    }

    /**
     * 异步回调(必须有,得发布到外网)
     *
     * @param unifiedorderUrl
     * @param requestXml
     * @return
     */
    @RequestMapping("/notifyUrl")
    public String notifyUrl(String unifiedorderUrl, String requestXml) {
        System.out.print("进入支付h5回调=====================");
        //如果没有加redirectUrl,就这这个接口处理订单信息
        //判断接受到的result_code是不是SUCCESS,如果是,则返回成功,具体业务具体分析
        // 通知微信.异步确认成功.必写.不然会一直通知后台.八次之后就认为交易失败了
        String resXml = "<xml>" + "<return_code><![CDATA[SUCCESS]]></return_code>" +
                "<return_msg><![CDATA[OK]]></return_msg>" + "</xml> ";
        return resXml; //或者 return "success";
    }
}

```
## pom

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.1.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.weixinpay</groupId>
    <artifactId>demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>demo</name>
    <description>Demo project for Spring Boot</description>

    <properties>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

        <!--微信支付SDK-->
        <dependency>
            <groupId>com.github.wxpay</groupId>
            <artifactId>wxpay-sdk</artifactId>
            <version>0.0.3</version>
        </dependency>
        <dependency>
            <groupId>org.apache.httpcomponents</groupId>
            <artifactId>httpclient</artifactId>
            <version>4.5.5</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.47</version>
        </dependency>
        <dependency>
            <groupId>com.google.zxing</groupId>
            <artifactId>core</artifactId>
            <version>3.3.0</version>
        </dependency>
        <dependency>
            <groupId>com.google.zxing</groupId>
            <artifactId>javase</artifactId>
            <version>3.0.0</version>
        </dependency>
        <dependency>
            <groupId>commons-io</groupId>
            <artifactId>commons-io</artifactId>
            <version>2.6</version>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>

        <finalName>demo</finalName>
        <resources>
            <resource>
                <directory>src/main/resources</directory>
                <filtering>true</filtering>
                <includes>
                    <include>**/*.properties</include>
                </includes>
            </resource>
            <resource>
                <directory>src/main/resources</directory>
                <filtering>false</filtering>
                <includes>
                    <include>**</include>
                </includes>
            </resource>
        </resources>
    </build>

</project>

```
## util

```
/**
 * @Description
 * @Date:03
 */
public class HttpUtil {

    public static String doPost(String url, String requestXml) {
        CloseableHttpClient httpClient = null;
        CloseableHttpResponse httpResponse = null;
        //创建httpClient连接对象
        httpClient = HttpClients.createDefault();
        //创建post请求连接对象
        HttpPost httpPost = new HttpPost(url);
        //创建连接请求对象,并设置连接参数
        RequestConfig requestConfig = RequestConfig.custom()
                .setConnectTimeout(15000)   //连接服务区主机超时时间
                .setConnectionRequestTimeout(60000) //连接请求超时时间
                .setSocketTimeout(60000).build(); //设置读取响应数据超时时间
        //为httppost请求设置参数
        httpPost.setConfig(requestConfig);
        //将上传参数放到entity属性中
        httpPost.setEntity(new StringEntity(requestXml, "UTF-8"));
        //添加头信息
        httpPost.addHeader("Content-type", "text/xml");
        String result = "";
        try {
            //发送请求
            httpResponse = httpClient.execute(httpPost);
            //从相应对象中获取返回内容
            HttpEntity entity = httpResponse.getEntity();
            result = EntityUtils.toString(entity, "UTF-8");
        } catch (IOException e) {
            e.printStackTrace();
        }
        return result;

    }

    /**
     * 获取IP地址
     *
     * @param request
     * @return
     */
    public static String getIpAddress(HttpServletRequest request) {
        String ip = request.getHeader("x-forwarded-for");
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getHeader("Proxy-Client-IP");
        }
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getHeader("WL-Proxy-Client-IP");
        }
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getHeader("HTTP_CLIENT_IP");
        }
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getHeader("HTTP_X_FORWARDED_FOR");
        }
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getRemoteAddr();
        }
        return ip;
    }
}

```
## properties
```
appid=wx12822223?22sss
mchId=149232323312333sss
weixinKey=34234234er2werwerwer
unifiedorderUrl=https://api.mch.weixin.qq.com/pay/unifiedorder
```
