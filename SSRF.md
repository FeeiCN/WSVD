# SSRF(Server-side Request Forge, 服务端请求伪造)。

### 漏洞简介
由攻击者构造的攻击链接传给服务端执行造成的漏洞，一般用来在外网探测或攻击内网服务。

### 直观例子
百度提供一个图片搜索功能，图片可以通过上传本地文件和填写图片地址两种方式。

### 漏洞代码
如果是填写图片地址，则百度图片搜索的后端实现就会先下载这个图片，然后再做其它处理，我们可以想象并简化下它的后端实现。

##### PHP版本
`SSRF_01.php`
```php
/**
 * Request service(Base file_get_contents)
 *
 * @author Feei <wufeifei@wufeifei.com>
 * @link   http://wufeifei.com/ssrf
 */
$url = $_GET['url'];
echo file_get_contents($url);
```

`SSRF_02.php`
```php
/**
 * Request service(Base cURL)
 *
 * @author Feei <wufeifei@wufeifei.com>
 * @link   http://wufeifei.com/ssrf
 */
function curl($url){  
    $ch = curl_init();
    curl_setopt($ch, CURLOPT_URL, $url);
    curl_setopt($ch, CURLOPT_HEADER, 0);
    curl_exec($ch);
    curl_close($ch);
}

$url = $_GET['url'];
curl($url);
```

`SSRF_03.php`
```php
/**
 * Request service(Base fsockopen)
 *
 * @author Feei <wufeifei@wufeifei.com>
 * @link   http://wufeifei.com/ssrf
 */
function request($host,$port,$link)
{
    $fp = fsockopen($host, $port, $err_no, $err_str, 30);
    $out = "GET $link HTTP/1.1\r\n";
    $out .= "Host: $host\r\n";
    $out .= "Connection: Close\r\n\r\n";
    $out .= "\r\n";
    fwrite($fp, $out);
    $contents='';
    while (!feof($fp)) {
        $contents.= fgets($fp, 1024);
    }
    fclose($fp);
    return $contents;
}
echo request($_GET['host'], $_GET['port'], $_GET['url']);
```

##### Java版本
- Java HttpURLConnection (java.net.HttpURLConnection)
- [Apache HttpClient (org.apache.http.client.HttpClient)](http://hc.apache.org/httpcomponents-client-ga/httpclient/apidocs/org/apache/http/impl/client/DefaultHttpClient.html)
- [HttpClient (org.apache.commonds.httpclient)](http://hc.apache.org/httpclient-%20-x/apidocs/index.html)

### 攻击复现

##### 服务探测
此时，我们构造一个探测请求：`http://10.11.2.1:80`填到图片地址输入框中，百度图片搜索后端会把这个请求当成图片去下载，由于我们填写的是内网IP，而百度图片搜索服务器肯定也在百度的内网中，如果服务存在，则能请求成功，此时就可以根据Request的返回时间可以判断对应端口上的服务是否开启，所以我们可以通过遍历所有内网IP，来判断百度内网的服务运行情况。

##### 进阶攻击
除了能探测，我们还可以通过构造攻击请求来实现对内网服务的攻击。

后端request服务一般都支持除HTTP/HTTP以外的协议，比如PHP中常用的curl request服务默认开启支持的协议：
```bash
$ curl -V
curl 7.47.1 (x86_64-apple-darwin15.3.0) libcurl/7.47.1 OpenSSL/1.0.2h zlib/1.2.8
Protocols: dict file ftp ftps gopher http https imap imaps pop3 pop3s rtsp smb smbs smtp smtps telnet tftp
Features: IPv6 Largefile NTLM NTLM_WB SSL libz TLS-SRP UnixSockets
```
支持file、dict、gopher等协议，所以我们可以利用这些协议来通过构造攻击请求去操作内网漏洞。

##### HTTP/HTTPS协议
- 网络服务探测
- ShellShock命令执行
- JBOSS远程Invoker war命令执行
- Java调试接口命令执行
- axis2-admin部署Server命令执行
- Jenkins Scripts接口命令执行
- Confluence SSRF
- Struts2一堆命令执行
- counchdb WEB API远程命令执行
- mongodb SSRF
- docker API远程命令执行
- php_fpm/fastcgi 命令执行
- tomcat命令执行
- Elasticsearch引擎Groovy脚本命令执行
- WebDav PUT上传任意文件
- WebSphere Admin可部署war间接命令执行
- Apache Hadoop远程命令执行
- zentoPMS远程命令执行
- HFS远程命令执行
- glassfish任意文件读取和war文件部署间接命令执行

##### file协议
可以用来直接读取文件内容
```
file:///etc/passwd
```

##### dict协议
可以用来操作Redis等
```
dict://10.11.2.1:6379/info
```
##### ftp、ftps
FTP匿名访问、爆破

##### tftp
UDP协议扩展

##### imap/imaps/pop3/pop3s/smtp/smtps
爆破邮件用户名密码

##### telnet
SSH/Telnet匿名访问及爆破

##### smb/smbs
SMB匿名访问及爆破

##### gopher协议
这个就是万金油协议了，什么都能干。能够将所有操作转成数据流，并将数据流一次发出去。所以可以用来探测内网的所有服务的所有漏洞。


### 漏洞一般存在的地方
	• 能够发起网络请求的地方
		○ 让你填写域名的地方
	• 从远程服务器请求资源
		○ 从URL上传图片
		○ 订阅RSS
	• 数据库内置功能
		○ Oracle
		○ MongoDB
		○ MSSQL
		○ Postgres
		○ CouchDB
	• 邮箱服务器收取其他邮箱邮件
		○ POP3/IMAP/SMTP
	• 文件处理、编码处理、属性处理
		○ FFmpeg
		○ ImageMagick
		○ Docx
		○ PDF
		○ XML

### 修复方案
同时做好以下三条即可杜绝SSRF漏洞：
##### 1. [此条必须做到]禁止非HTTP、HTTPS协议的使用，使SSRF危害从高危降到低危；
##### 2. 禁止请求域名的301跳转（杜绝使用正常HTTP/HTTPS请求301跳转到攻击请求的方式）；
##### 3. 给请求域名设置白名单；

`PHP cURL`
```php
/**
 * Request service(Base cURL)
 *
 * @author Feei <wufeifei@wufeifei.com>
 * @link   http://wufeifei.com/ssrf
 */
function curl($url){
    $ch = curl_init();
    curl_setopt($ch, CURLOPT_URL, $url);
    curl_setopt($ch, CURLOPT_HEADER, 0);
    /**
     * 1. 增加此行限制只能请求HTTP/HTTPS的服务
     * @link https://curl.haxx.se/libcurl/c/CURLOPT_PROTOCOLS.html
     */
    curl_setopt($ch, CURLOPT_PROTOCOLS, CURLPROTO_HTTP | CURLPROTO_HTTPS);  
    /**
     * 2. 增加此行禁止301跳转
     * @link https://curl.haxx.se/libcurl/c/CURLOPT_FOLLOWLOCATION.html
     */
    curl_setopt($ch, CURLOPT_FOLLOWLOCATION, 0);  
    curl_exec($ch);
    curl_close($ch);
}

$url = $_GET['url'];
/**
 * 3. 请求域名白名单
 */
 $white_list_urls = [
     'www.wufeifei.com',
     'www.mogujie.com'
 ]
 if (in_array(parse_url($url)['host'], $white_list_urls)){
     echo curl($url);
 } else {
     echo 'URL not allowed';
 }
```

`Java:HttpURLConnection`
```java
/**
 * 1. 限制允许HTTP/HTTPS协议
 */
if(!url.getProtocol().startsWith("http"))
    throw new Exception();
/**
 * 3. 请求域白名单
 */
InetAddress inetAddress = InetAddress.getByName(url.getHost());
if(inetAddress.isAnyLocalAddress() || inetAddress.isLoopbackAddress() || inetAddress.isLinkLocalAddress())
    throw new Exception();
HttpURLConnection conn = (HttpURLConnection)(url.openConnection());
/**
 * 2. 禁止301跳转
 */
conn.setInstanceFollowRedirects(false);
conn.connect();
IOUtils.copy(conn.getInputStream(), out);
```

### 参考资料
- [SSRF到GET SHELL](http://wufeifei.com/ssrf/)

### 本章作者

```
转载请注明作者及本文章链接
Feei <wufeifei[at]wufeifei.com>
```
