# WAVR
Web application vulnerability repair, Web应用漏洞修复方案.

在大部分公司里是没有专职的安全工程师，而互联网上的大部分的漏洞文章都是侧重于怎么去利用的；
对于漏洞修复相关资料少之又少，仅有的几篇可能还只是站在攻击的角度讲的防御，按照这样的修复方式反而还会导致二次安全隐患。

WAVR是为了让所有公司都能找到最准确、全面、权威的漏洞修复方案。

## 安装
```bash
git clone https://github.com/wufeifei/WAVR
cd WAVR
[sudo] pip install mkdocs
mkdocs serve
```

## 漏洞列表

|中文名|英文名|修复方案|
|---|---|---|
|服务端请求伪造|SSRF（Server-side Request Forge）|[WAVR-SSRF](https://github.com/wufeifei/WAVR/blob/master/SSRF.md)|
|硬编码密码|Hard-coded Password|[WAVR-HP](https://github.com/wufeifei/WAVR/blob/master/Hard-coded_password.md)|
|跨站请求伪造|CSRF（Server-side Request Forge）|[WAVR-CSRF](https://github.com/wufeifei/WAVR/blob/master/CSRF.md)|
