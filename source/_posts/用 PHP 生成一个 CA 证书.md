---
title: 用php生成一个自己的CA证书并签发服务器证书
date: 2019-01-10 11:43:43
tags: 
- php
- openssl
categories: php
---

网上都是直接使用openssl命令生成CA证书，看着命令里面超多参数。一下子还是非常懵的。这里主要用php的openssl_*函数实现一个。

## 说明

有一个概念挺重要的。openssl.cnf这个文件非常的重要。如果想真正的做一个CA证书以前签发证书。需要熟悉openssl.cnf的各种配置。也可以在使用的时候。慢慢完善你的openssl配置

### 1.生成你的CA私钥对

```php
// 生成私钥对, 可以不带参数，如果不带则会使用openssl.cnf的配置生成
// 也可以指定另外的openssl.cnf, 这里把系统默认的openssl.cnf拷贝到当前目录下
$key = openssl_pkey_new([
	"config" => "./openssl.cnf",
	'private_key_bits' => 1024
]);

//可以将生成的私钥导出到文件中，下次直接读取即可
openssl_pkey_export_to_file($key, "cakey.pem");

```

PHP7.0支持了ECC(椭圆加密算法)，可以添加两个参数生成
大概吹一下ECC算法

> 抗攻击性强
> CPU 占用少
> 内容使用少
> 网络消耗低
> 加密速度快
> balabala ...

```PHP
$cakey = openssl_pkey_new([
	"config" => "./openssl.cnf",
	"private_key_type" => OPENSSL_KEYTYPE_EC,
	"curve_name" => 'prime256v1'
]);

openssl_pkey_export_to_file($res, "cakey.pem");
```

### 2.现在就可以生成你的CA根证书

CA根证书其实是使用自己的私钥对证书请求自签名得到的

```php
// 生成证书请求, 需要包括一些信息。更多的参数也可以了解openssl.cnf
$csr = openssl_csr_new([
	"countryName" => "CN",
	"stateOrProvinceName" => "GuangDong",
	"organizationName" => "BeikeSama",
	"commonName" => "BeikeSama Root CA 2019",
], $cakey);

// 接下来就可以用自己的私钥自签证书请求即可， 第二个参数为null则是自签名
$x509 = openssl_csr_sign(
	$csr, null, $cakey, $days=365, ['digest_alg' => 'sha384']
);

// 导出到文件中，这个就是根证书了，可以给其他小伙伴安装到电脑上或者手机上了
// Windows 注意选择安装到 受信任的根证书颁发机构
openssl_x509_export_to_file($x509, "beikesama-ca.crt");
```

### 3. 最后就是用上面的证书签发子证书

下面基本就是重复上面的步，变一下参数就好了

```php
// 还是生成私钥对，这里是给服务器端用的
// 一般来说的这一步是需要你签发证书的小伙伴生成的
$server_key = openssl_pkey_new([
    "config" => "./openssl.cnf",
    'private_key_bits' => 1024
]);

// 导出到文件中
openssl_pkey_export_to_file($key, "server_key.pem");

// 生成证书请求, 需要包括一些他的信息。然后你可以根据他的信息，对他验证，是不是你的小伙伴本人, commonName 一般是他的域名
$csr = openssl_csr_new([
	"countryName" => "CN",
	"stateOrProvinceName" => "GuangDong",
	"organizationName" => "BeikeSama",
	"commonName" => "beikesama.com",
], $server_key);

```

然后你的小伙伴把他的证书请求发给你

```php
// 通过他提供的信息，判断他是不是你的小伙伴
// 使用你的证书和私钥对他的证书请求签发就可以了

$cacrt = file_get_contents("beikesama-ca.crt");

// 还有一点需要注意 x509_extensions => v3_req
// 这个是openssl.cnf里的一个section，用来生成子证书的，这个在下面说，这里先这么写就好了
// 指定一下签名算法, 默认的话应该会是sha1。
// 不然Chrome还是会因为签名算法太弱，提示不安全。
$x509 = openssl_csr_sign(
	$csr, $cacrt, $cakey, $days=365, [
		"config" => './openssl.cnf',
		"x509_extensions" => "v3_req",
		'digest_alg' => 'sha384'
	]
);

// 导出证书，这个就可以填写到Nginx里面使用咯
openssl_x509_export_to_file($x509, "beikesama.com.crt");
```

### 4.关于上面的openssl.cnf

这里需要在openssl.cnf里面找到v3_req这个section添加一个subjectAltName

看起来像这样

```ini

[ v3_req ]

# Extensions to add to a certificate request

basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment

subjectAltName          = @alt_names

[ alt_names ]
DNS.1 = beikesama.com
```

DNS.1 就是小伙伴的域名了。这个相当于签发证书里面的扩展信息，浏览器会根据这个扩展信息判断该域名是不是被授权的。可以写多个。