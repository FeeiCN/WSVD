# CSRF(Cross-site request forgery, 跨站请求伪造)，也称为XSRF。

### 漏洞简介
挟持用户在当前已登陆的Web应用程序上执行非本意的操作。

### 直观例子
淘宝店铺页面提供收藏店铺功能。

### 漏洞代码
我们可以简化下收藏店铺的后端实现。

```php
<?php
/**
 * Favorite shop for CSRF Example
 *
 * @author Feei <wufeifei[at]wufeifei.com>
 * @link   https://github.com/wufeifei/WAVR/blob/master/CSRF.md
 */
 $shop_id = $_GET['shop_id'];
 if (favorite_shop($shop_id)) 
 {
    echo "Favorite success!";
 }else {
 	echo "Favorite failed!";
 }
```

### 漏洞复现
此时可以构造一个收藏店铺的GET请求：
`http://shop.taobao.com/favorite_shop?shop_id=123`

这个链接丢给其他人，只要他的淘宝是已经登陆过的，那么就会自动关注你制定的店铺。

如果想扩大影响，可以将这个链接放入正常的站点的请求中，比如放入自己博客中。
博客内嵌入`<img src="http://shop.taobao.com/favorite_shop?shop_id=123">`，只要访问你博客的用户都将自动关注你指定的店铺。

### CSRF漏洞修复最佳实践

#### 1. 理解GET/POST/REQUEST的区别

#### 2. Referer限制
在后端应用框架层或更上层（Nginx、WAF等）给所有接口增加请求Refere白名单，只有自己业务的根域及信任的域可以直接调用接口。此举将CSRF危害降低到域内范围。

注意：Referer验证时禁止使用字符串匹配或包含的方式来判断，应当使用以下推荐方式防止被绕过。

```java
import java.net.*;
import java.io.*;

/**
 * Example for WAVR CSRF
 *
 * @author Feei <wufeifei[at]wufeifei.com
 * @link   https://github.com/wufeifei/WAVR/blob/master/CSRF.md
 */
public class ParseURL {
  public static void main(String[] args) throws Exception {

    URL aURL = new URL("http://wufeifei.com:80/wavr/csrf/tutorial/index.html?name=networking#MIAODIAN");

    System.out.println("protocol = " + aURL.getProtocol()); //http
    System.out.println("authority = " + aURL.getAuthority()); //wufeifei.com:80
    // 使用.getHost()来获取根域进行白名单比对
    System.out.println("host = " + aURL.getHost()); //wufeifei.com
    System.out.println("port = " + aURL.getPort()); //80
    System.out.println("path = " + aURL.getPath()); //  /wavr/csrf/tutorial/index.html
    System.out.println("query = " + aURL.getQuery()); //name=networking
    System.out.println("filename = " + aURL.getFile()); ///wavr/csrf/tutorial/index.html?name=networking
    System.out.println("ref = " + aURL.getRef()); //MIAODIAN
  }
}
```

```php
<?php
/**
 * Example for WAVR CSRF
 * 
 * @author Feei <wufeifei[at]wufeifei.com
 * @link   https://github.com/wufeifei/WAVR/blob/master/CSRF.md
 */
$url_arr = parse_url("http://wufeifei.com:80/wavr/csrf/tutorial/index.html?name=networking#MIAODIAN");
print_r($url_arr);
Array
(
    [scheme] => http
    [host] => wufeifei.com
    [port] => 80
    [path] => /wavr/csrf/tutorial/index.html
    [query] => name=networking
    [fragment] => MIAODIAN
)

// 使用$url_arr['host']来获取根域进行白名单比对
print_r($url_arr['host']);
```

#### 3. CSRF-Token
每次表单渲染完将会有一个隐藏Token字段（CSRF-Token），该Token字段将随每次表单一起提交，后端处理表单前将先验证Token是否合法再处理表单业务。

### 附： CSRF-Token安全组件技术方案设计
如果是单应用且应用使用了常用框架，则框架内默认会自带CSRF-Token实现，无需另行开发。比如Python的Flask、Django等；PHP的Laravel、Kohana等；Java的Spring MVC等。

如果是小应用，则使用常见的SESSION Token服务即可。

这里主要介绍下多应用、多框架下的CSRF-Token安全组件。

提供根据UUID生成和验证Token的组件服务：

Token生成服务：
- 根据UUID+SALT使用HMAC-Sha1算法生成Token
- 将Token作为Key储存在Cache中，将UUID作为Value存储在Cache中，并设置好失效期
- 返回Token给前台页面上

Token验证服务：
- 检查Cache中是否有Key为该Token的数据，没有则返回验证失败（状况：过期或伪造的Token）
- 检查Cache中对应的值是否为该UUID，不是则返回验证失败（状况：Token和UUID不对应）
- 检查Key和Value都通过则返回验证成功，没有则返回失败

核心点
- 生成Token要快：Token的作用是为了让每次请求都唯一，只要Token不能被猜到就可以，不需要太复杂的算法；
- 失效时间：一个Token在非关键业务上可以多次使用，但一定需要失效时间（默认建议1小时）；
- 附属作用一：核心业务的Token是一次使用即失效，可以起到解决请求重放的问题；
- 附属作用二：解决JSONP随意调用的问题；