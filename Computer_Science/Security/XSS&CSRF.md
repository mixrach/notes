# CSRF

> 前提： CSRF只有在浏览器留存的用户登录态存在泄漏风险时才可能发生。

- 保ME：不存在登录态，只有用户openID，基本不存在openID泄漏的问题，而且保ME中信息不敏感。
- C2C：前后端分离，存在攻击可能
- B2C： 后端模板渲染，但不许登录，理论上不存在攻击可能，但是可以通过token防刷接口

## 解决方案：

### 1. 针对前后端分离

#### Refer + Token



```sequence

participant 前端
participant Zuul
participant server
前端->Zuul: 验证登录?
Zuul->前端: 登录成功，cookie携带token与salt
前端->Zuul: 取出Cookie中的token值写入Header中
Note over Zuul: 验证Refer域名
Zuul-->前端:验证失败，403 forbidden
Note over Zuul: R = Decrypt(salt);
Note over Zuul:MD5（secret + pin + R） == tokenValue ？
Zuul-->前端: 认证失败：403 forbidden
Zuul->server:认证成功：路由放行

```

#### 备注

- Zuul集群部署时，secret值得更新必须能够同步更新
- salt = Encrypted（R）；R = random string
- token =  MD5（secret+Pin+ R) ；

## 2. 后端渲染

#### Session + hidden form field [不推荐]

用户请求表单的时候，页面Session生成一个临时的token值，将这个token插入Form的一个hidden域中，用户提交表单时验证hidden域中的值与Session中的token值。token使用一次后即失效，需再次生成token。

缺点

- 分布式情况Session需共享
- token值单一，与后续的页面预渲染优化冲突
- 如果采用token放入redis中，需要考虑token的过期时间

#### Refer + Double-submitted cookie [推荐]

当用户访问一个站点，这个站点需要生成一个伪随机数，并写入用户cookie中。后端服务器接收请求时期望所有表单中以及cookie中都有此随机数。由于浏览器的同源策略，攻击者无法读取或者修改cookie中的值，攻击者必须能够猜测到此伪随机值才能进行攻击。

```sequence
participant server
participant 前端
前端-> server:登录验证
Note over server:验证成功
Note over server:生成随机token
server->前端: 随机值写入cookie（cookie.token）（过期时间与用户登录状态一致）
Note over 前端: 填写表单
Note over 前端: JS函数提交表单，并添加域，token=cookie.token
前端->server: 表单提交
Note over server:拦截器验证Refer
server-->前端:验证失败，403 forbidden
Note over server:token == cookie.token?
server-->前端:验证失败，403 forbidden
Note over server:验证成功
Note over server:提交成功
```

# XSS

## 几条原则

>RULE #1 - HTML Escape Before Inserting Untrusted Data into HTML Element Content
>
>RULE #2 - Attribute Escape Before Inserting Untrusted Data into HTML Common Attributes
>
>RULE #3 - JavaScript Escape Before Inserting Untrusted Data into JavaScript Data Values
>
>RULE #4 - CSS Escape And Strictly Validate Before Inserting Untrusted Data into HTML Style Property Values
>
>RULE #5 - URL Escape Before Inserting Untrusted Data into HTML URL Parameter Values
>
>RULE #6 - Sanitize HTML Markup with a Library Designed for the Job
>
>RULE #7 - Prevent DOM-based XSS

| Data Type | Context                                  | Code Sample                                                  | Defense                                                      |
| --------- | ---------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| String    | HTML Body                                | \<span>UNTRUSTED DATA\</span>                                | [HTML Entity Encoding](https://www.owasp.org/index.php/XSS_(Cross_Site_Scripting)_Prevention_Cheat_Sheet#RULE_.231_-_HTML_Escape_Before_Inserting_Untrusted_Data_into_HTML_Element_Content) |
| String    | Safe HTML Attributes                     | \<input type="text" name="fname" value="UNTRUSTED DATA">     | [Aggressive HTML Entity Encoding](https://www.owasp.org/index.php/XSS_(Cross_Site_Scripting)_Prevention_Cheat_Sheet#RULE_.232_-_Attribute_Escape_Before_Inserting_Untrusted_Data_into_HTML_Common_Attributes)Only place untrusted data into a whitelist of safe attributes (listed below).Strictly validate unsafe attributes such as background, id and name. |
| String    | GET Parameter                            | \<a href="/site/search?value=UNTRUSTED DATA">clickme\</a>    | [URL Encoding](https://www.owasp.org/index.php/XSS_(Cross_Site_Scripting)_Prevention_Cheat_Sheet#RULE_.235_-_URL_Escape_Before_Inserting_Untrusted_Data_into_HTML_URL_Parameter_Values) |
| String    | Untrusted URL in a SRC or HREF attribute | <a href="UNTRUSTED URL">clickme</a> \<iframe src="UNTRUSTED URL" /> | Canonicalize inputURL ValidationSafe URL verificationWhitelist http and https URL's only ([Avoid the JavaScript Protocol to Open a new Window](https://www.owasp.org/index.php/Avoid_the_JavaScript_Protocol_to_Open_a_new_Window))Attribute encoder |
| String    | CSS Value                                | \<div style="width: UNTRUSTED DATA;">Selection\</div>        | [Strict structural validation](https://www.owasp.org/index.php/XSS_(Cross_Site_Scripting)_Prevention_Cheat_Sheet#RULE_.234_-_CSS_Escape_And_Strictly_Validate_Before_Inserting_Untrusted_Data_into_HTML_Style_Property_Values)CSS Hex encodingGood design of CSS Features |
| String    | JavaScript Variable                      | \<script>var currentValue='UNTRUSTED DATA';\</script> \<script>someFunction('UNTRUSTED DATA');\</script> | Ensure JavaScript variables are quotedJavaScript Hex EncodingJavaScript Unicode EncodingAvoid backslash encoding (\" or \' or \\) |
| HTML      | HTML Body                                | \<div>UNTRUSTED HTML\</div>                                  | [HTML Validation (JSoup, AntiSamy, HTML Sanitizer)](https://www.owasp.org/index.php/XSS_(Cross_Site_Scripting)_Prevention_Cheat_Sheet#RULE_.236_-_Use_an_HTML_Policy_engine_to_validate_or_clean_user-driven_HTML_in_an_outbound_way) |
| String    | DOM XSS                                  | \<script>document.write("UNTRUSTED INPUT: " + document.location.hash);\<script/> | [DOM based XSS Prevention Cheat Sheet](https://www.owasp.org/index.php/DOM_based_XSS_Prevention_Cheat_Sheet) |



> # 没有银弹！！！

## XSS 可行方案

### 针对前后端分离

- 方案一： 如果确认请求数据只是String 插入HTML Body, 则可以实现统一的AJAX返回值处理
- 方案二： 对于应用中可能涉及到的脚本插入，HTML插入等，借鉴Angular解决方案

### 针对后端渲染

- 方案一：暴力方法，对于模板中每一个可能存在攻击的位置（html，src）等内容进行转义。可以使用Velocity提供的`EscapeTool`或者OWSAP提供的`JavaEncoder`结合Velocity的Directive。
- 方案二：（如果只是escape Html）则可以添加Velocity的`ReferenceInsertionHandler`，在渲染前截取输出内容进行转义，并针对插入HTML的需求定义Directive

### XSS最终方案

前后端分离：方案一Ins_

后端渲染：方案二