# Hard-coded password

### 漏洞简介
Hard-coded password(硬编码密码)。
硬编码密码是将非加密密码（纯文本）或者密钥写在源代码中。

硬编码密码问题会在源代码泄露的情况下造成危害：
- Web目录未限制.git文件夹的访问
- 项目代码公开在Github等代码托管平台上
- 私有Gitlab遭到爆破或者漏洞
- 张贴代码片段到邮件或即时通讯工具上
- etc.

造成的影响影响持久且难以恢复，比如：

- 外部可以直接通过密码认证
- 将硬编码密码加入字典中爆破其它系统
- etc.


### 密码定义
有鉴权作用的（哪怕不能直接利用）都算作密码口令。
- 邮箱等账号密码
- DB/Redis/SSH/FTP/Rsync等服务密码
- Key或Token
- 证书文件或密钥文件
- etc.

### 硬编码密码减轻或优化方法

- 询问用户密码：对于供用户使用的Web应用程序，密码应当询问用户，而非写死；
- 保存在一个独立文件中：保存在一个文件是比较规范的操作，避免你在源码中多处硬编码。并且为该文件设立一个严格的文件权限，并不允许传到代码版本控制系统上；
- 加密：通过使用3DES、RSA、Rijndael等算法来加密密码并放入一个独立文件中，同时加密密钥自己留存；
- 保存在数据库：密码储存在数据库也是一个比较好的方式；
- 使用令牌系统：当硬编码密码在脚本中，可以使用AFS或者Kerberos令牌；
- Hash用户的密码：当在客户端储存密码时（理论上不允许），请使用无法反解的Hash算法储存；
- 配置中心：自建密码配置中心用来分发密码等密码管理工作；

### 通用硬编码密码最佳实践
##### 1. 将源码中密码加密后存储在独立成配置文件中；
通过Cobra等代码审计系统将所有项目中的硬编码密码问题找出后，使用对称加密算法（AES）将密码加密后存储在独立的配置文件中，使用时读取配置文件中的密码解密后使用。
##### 2. 将配置文件从VCS中删除后放入配置管理系统；
将配置文件[从VCS（版本控制系统，比如git/svn等）无痕删除](https://help.github.com/articles/remove-sensitive-data/)，导入配置管理系统。
##### 3. 配置管理系统来管理和发布配置，并保障配置储存安全；
由配置管理系统负责加密储存配置信息到独立数据库中；Java：在应用发布时由发布系统去请求配置管理系统获取配置信息并打包至应用中；PHP：由配置管理系统独立发布配置文件至应用配置服务器指定路径，并做好配置文件的只读权限；

#### 附: AES（128 CBC NoPadding）对称加密实现参考

##### Java
```java
import javax.crypto.Cipher;
import javax.crypto.spec.IvParameterSpec;
import javax.crypto.spec.SecretKeySpec;
import sun.misc.BASE64Decoder;

/**
 * AES Example
 *
 * @author Feei <wufeifei[at]wufeifei.com>
 * @link   https://github.com/wufeifei/WAVR/blob/master/Hard-coded_password.md
 */
public class Encryption
{

    public static void main(String args[]) throws Exception {
        System.out.println(encrypt());
        System.out.println(desEncrypt());
    }
    
    public static String encrypt() throws Exception {
        try {
            String data = "WAVR_HP_AES";
            // key/iv 建议随机生成，并储存至独立配置文件中
            String key = "1234567812345678";
            String iv = "1234567812345678";
 
            Cipher cipher = Cipher.getInstance("AES/CBC/NoPadding");
            int blockSize = cipher.getBlockSize();
 
            byte[] dataBytes = data.getBytes();
            int plaintextLength = dataBytes.length;
            if (plaintextLength % blockSize != 0) {
                plaintextLength = plaintextLength + (blockSize - (plaintextLength % blockSize));
            }
 
            byte[] plaintext = new byte[plaintextLength];
            System.arraycopy(dataBytes, 0, plaintext, 0, dataBytes.length);
 
            SecretKeySpec keyspec = new SecretKeySpec(key.getBytes(), "AES");
            IvParameterSpec ivspec = new IvParameterSpec(iv.getBytes());
 
            cipher.init(Cipher.ENCRYPT_MODE, keyspec, ivspec);
            byte[] encrypted = cipher.doFinal(plaintext);
 
            return new sun.misc.BASE64Encoder().encode(encrypted);
 
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
    }
 
    public static String desEncrypt() throws Exception {
        try
        {
            String data = "O6wI6+I9ejeN4P7IR5lGGg==";
            // key/iv 建议随机生成，并储存至独立配置文件中
            String key = "1234567812345678";
            String iv = "1234567812345678";
 
            byte[] encrypted1 = new BASE64Decoder().decodeBuffer(data);
 
            Cipher cipher = Cipher.getInstance("AES/CBC/NoPadding");
            SecretKeySpec keyspec = new SecretKeySpec(key.getBytes(), "AES");
            IvParameterSpec ivspec = new IvParameterSpec(iv.getBytes());
 
            cipher.init(Cipher.DECRYPT_MODE, keyspec, ivspec);
 
            byte[] original = cipher.doFinal(encrypted1);
            String originalString = new String(original);
            return originalString;
        }
        catch (Exception e) {
            e.printStackTrace();
            return null;
        }
    }
}
```

##### PHP
```php
<?php
/**
 * AES Example
 * 
 * @author Feei <wufeifei[at]wufeifei.com>
 * @link   https://github.com/wufeifei/WAVR/blob/master/Hard-coded_password.md
 */
// key/iv 建议随机生成，并储存至独立配置文件中
date_default_timezone_set('Asia/Shanghai');
$privateKey = "1234567812345678";
$iv         = "1234567812345678";
$data       = "WAVR_HP_AES";
 
//加密
$encrypted = mcrypt_encrypt(MCRYPT_RIJNDAEL_128, $privateKey, $data, MCRYPT_MODE_CBC, $iv);
echo(base64_encode($encrypted).PHP_EOL);
 
//解密
$encryptedData = base64_decode("O6wI6+I9ejeN4P7IR5lGGg==");
$decrypted = mcrypt_decrypt(MCRYPT_RIJNDAEL_128, $privateKey, $encryptedData, MCRYPT_MODE_CBC, $iv);
echo($decrypted);
?>
```