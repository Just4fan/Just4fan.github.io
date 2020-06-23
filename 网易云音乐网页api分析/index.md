# 网易云音乐网页API分析


先随便选一个接口分析一下，就选评论接口吧。

### 1.获取评论 ###

进入Chrome开发者模式可以看到

![avatar](https://s2.ax1x.com/2019/06/02/VGR8Yj.png)
![avatar](https://s2.ax1x.com/2019/06/02/VGWc5Q.png)

这个就是获取评论的接口，请求方式是POST，请求体有两个参数params和encSecKey，在请求的Initiator可以看到发出请求的代码在core_xxxx.js这个文件，代码肯定经过混淆，先下载到本地格式化一下。在js文件中搜索encSecKey，找到生成参数的相关代码段，可以看到参数由asrsea这个函数生成。

```javascript
var bYc0x = window.asrsea(JSON.stringify(i8a), bkY9P(["流泪", "强"]), bkY9P(VM5R.md), bkY9P(["爱心", "女孩", "惊恐", "大笑"]));
e8e.data = k8c.cz9q({
    params: bYc0x.encText,
    encSecKey: bYc0x.encSecKey
})
```

继续在js文件中搜索这个函数，可以看到生成参数的相关函数：

```javascript
function a(a) {
    var d, e, b = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789",
        c = "";
    for (d = 0; a > d; d += 1) e = Math.random() * b.length,
    e = Math.floor(e),
    c += b.charAt(e);
    return c
}
//生成encText，即params
function b(a, b) {
    var c = CryptoJS.enc.Utf8.parse(b),                     //密钥
        d = CryptoJS.enc.Utf8.parse("0102030405060708"),    //初始化向量
        e = CryptoJS.enc.Utf8.parse(a),                     //数据
        f = CryptoJS.AES.encrypt(e, c, {
            iv: d,
            mode: CryptoJS.mode.CBC
        });
    return f.toString()
}
//生成encSecKey
function c(a, b, c) {
    var d, e;
    return setMaxDigits(131),
    d = new RSAKeyPair(b, "", c),
    e = encryptedString(d, a)
}
//spider 加密函数
function d(d, e, f, g) {
    var h = {},
        i = a(16);  //随机生成一个由a~z,A~Z,0~9组成的长度为a的字符串（AES密钥）
    return h.encText = b(d, g),
    h.encText = b(h.encText, i),
    h.encSecKey = c(i, e, f),
    h
}
function e(a, b, d, e) {
    var f = {};
    return f.encText = c(a + e, b, d),
    f
}
window.asrsea = d,
window.ecnonasr = e
```

params的生成过程比较简单，在函数d中可以看到params由两次调用函数b得到：函数b是AES加密的过程，参数a是要加密的数据，参数b是密钥，params经过两次AES加密得到，通过调试可以看到第一次加密：

```javascript
h.encText = b(d, g)
```

参数g是固定的：

> 0CoJUm6Qyw8W8jud

而参数d则是请求参数构成的json字符串，在上面的请求中是：

> {"rid":"R_SO_4_1361988914","offset":"0","total":"true","limit":"20","csrf_token":""}

offset代表获取的第一条评论的偏移量，limit是获取的评论数。

第二次加密是使用随机生成的密钥（长度为16）加密第一次加密生成的数据。为了方便测试，在原网页获取随机生成的某密钥以及用这个密钥加密得到的params和encSecKey：

key:

> 6lauqIqaJelwkFKM

params:

> 1NGJ2rghqo1ToCy5r27rEDqRCNEZxxKk2BvmOQoud6GTNfPZKGRJVFmqfJWeAXYQi4V/++6HKpDjFDfB+TZZ+cmDQF2ywgL9w5Ow8r+mtLG95D7rfUfRkT7GiTn4KMCt+mIM9dtgtIoKXSZ77407F/ZI/GCfslievQ0IfnBAR8SoKQmz8XaQgC1Lxo93VxQ4

encSecKey:

> 2d1a892a0d7b3101eb00936c4915b2541bc378b58a5ccf6ab3ca939da1fd1fcee7b7456feef0a0d9115c1b5aad8aa64674df8249597e9e74ab66cb5fa84a624f174ed59e786fef748528fff098fa749c11aa06d7d86b7a41d917277efb3f14f8b974ddb2cb5df638ec36d3de8c2f02335cb1990b150a8deafc186c2007dea21c

其中AES加密的填充方式是pkcs#7padding，块大小是16 Bytes，加密后的数据默认使用base64编码，根据pkcs#7padding的规则，需要在数据后面添加(16 - n%16)个byte（无论n的长度是否是16的整数倍），且byte的值也为(16 - n%16)。

```python
from Crypto.Cipher import AES
import base64

def pkcs7_padding(data):
	n = 16 - len(data) % 16
	data += (chr(n) * n).encode()
	return data

key = '0CoJUm6Qyw8W8jud'.encode('utf-8')
mode = AES.MODE_CBC
iv = '0102030405060708'.encode('utf-8')

aes = AES.new(key, mode, iv)

data = '{"rid":"R_SO_4_1361988914","offset":"0","total":"true","limit":"20","csrf_token":""}'.encode('utf-8')
data = pkcs7_padding(data)

endata = aes.encrypt(data)
endata = base64.b64encode(endata)

key = '6lauqIqaJelwkFKM'.encode('utf-8')

aes = AES.new(key, mode, iv)

data = pkcs7_padding(endata)

endata = aes.encrypt(data)
endata = base64.b64encode(endata)

print(endata)
```

得到的数据与请求中的一致。

再分析一下encSecKey的生成过程，在函数d中可以看到encSecKey由函数c生成：函数c将随机生成的密钥（就是上面生成params时AES加密的密钥）经过RSA加密得到encSecKey。
关于RSA加密的过程，源代码在这里可以看到:http://www.ohdave.com/rsa

RSAKeyPair包括encryptionExponent,decryptionExponent,modulus（分别对应代码中RSAKeyPair函数的三个参数，根据RSA加密的过程：公钥包含两个大整数，n和e，假设要加密的数据是m，则加密后的数据$c ≡ m^e \ (mod \ n)$,这里的e就是encryptionExponent，n就是modulus。

e和n两个数应该是固定的（公钥应该不会频繁更换吧），在RSAKeyPair函数打断点得到e和n的值分别是：

> 010001

和

> 00e0b509f6259df8642dbc35662901477df22677ec152b5ff68ace615bb7b725152b3ab17a876aea8a5aa76d2e417629ec4ee341f56135fccf695280104e0312ecbda92557c93870114af6c9d05c4f7f0c3685b7a46bee255932575cce10b424d813cfe4875d3e82047b97ddef52741d546b8e289dc6935b3ece0462db0a22b8e7

一开始我尝试直接对数据进行加密：

```python
import rsa
import binascii

rsa_n = int("00e0b509f6259df8642dbc35662901477df22677ec152b5ff68ace615bb7b725152b3ab17a876aea8a5aa76d2e417629ec4ee341f56135fccf695280104e0312ecbda92557c93870114af6c9d05c4f7f0c3685b7a46bee255932575cce10b424d813cfe4875d3e82047b97ddef52741d546b8e289dc6935b3ece0462db0a22b8e7", 16)
rsa_e = int("010001", 16)

key = rsa.PublicKey(rsa_n, rsa_e)
data = "6lauqIqaJelwkFKM".encode()

encrypted_data = rsa.encrypt(data, key)
hex_str = binascii.b2a_hex(encrypted_data)

print(hex_str)
```

尝试请求接口，但是没有返回数据，猜测跟数据转换成的整数有关，回到js代码中加密的代码段看看：

```javascript
function encryptedString(a, b) {
    for (var f, g, h, i, j, k, l, c = new Array, d = b.length, e = 0; d > e;) c[e] = b.charCodeAt(e),
    e++;
    for (; 0 != c.length % a.chunkSize;) c[e++] = 0;
    for (f = c.length, g = "", e = 0; f > e; e += a.chunkSize) {
        for (j = new BigInt, h = 0, i = e; i < e + a.chunkSize; ++h) {
            j.digits[h] = c[i++]
            j.digits[h] += c[i++] << 8
        }
        //j是待加密数据转换后的整数
        k = a.barrett.powMod(j, a.e),
        l = 16 == a.radix ? biToHex(k) : biToString(k, a.radix),
        g += l + " "
    }
    return g.substring(0, g.length - 1)
}
```

在本地测试输出j的值：

```javascript
console.log(biToHex(j))
```

对这个值进行加密：

```python
hex_str = '4d4b466b776c654a6171497175616c36'
rsa_m = int(hex_str, 16)

rsa_c = rsa.core.encrypt_int(rsa_m, rsa_e, rsa_n)

print(hex(rsa_c))
```

这次得到的数据跟请求中的参数一致（RSA加密生成的数据可能每次都不一样，跟填充方式有关），查阅RSA相关的资料知道算法是在固定长度的块上面进行操作，如果数据长度大于块大小要分割，小于要填充，根据上面的代码，这里的填充方式是直接在后面补'\0'，加密时对每块进行加密，生成的数据之间用空格隔开，至于块大小怎么确定暂时不知道，先不管。
为了了解数据是怎么转换成整数的，需要了解BigInt的结构，通过调试可以知道BigInt包含一个数组，观察上面的代码可以知道数据转换成BigInt的过程是把每两个的字符的ascii码值逆序存放到数组的一个元素中，例如字符串："cd",写成二进制的形式是：

> 0110 0011 / 0010 0100

放到BigInt中是：

> 0010 0100 0110 0011

接下来看看BigInt是怎么转换成整数的，找到BigInt转Hex的函数（因为这个函数最简单2333）：

```javascript
function reverseStr(a) {
    var c, b = "";
    for (c = a.length - 1; c > -1; --c) b += a.charAt(c);
    return b
}
function digitToHex(a) {
    var b = 15,
        c = "";
    for (i = 0; 4 > i; ++i) c += hexToChar[a & b],
    a >>>= 4;
    return reverseStr(c)
}
function biToHex(a) {
    var d, b = "";
    for (biHighIndex(a), d = biHighIndex(a); d > -1; --d) b += digitToHex(a.digits[d]);
    return
```

这个函数逆序对BigInt的数组元素进行如下操作：从低位开始每4个二进制位转对应的16进制，并逆序输出。
所以数据转整数的整个过程其实就是把数据以小端方式存储（跟直接加密的区别就在于大端和小端），例如字符串"cdef"：

0110 0011 / 0010 0100 / 0110 0101 / 0110 0110

转换成小端：

0110 0110 / 0110 0101 / 0010 0100 / 0110 0011

转换成16进制就是：

0x66656463

测试一下：

```python
table = ['0', '1', '2', '3', '4', '5', '6', '7', '8', '9', 'a', 'b', 'c', 'd', 'e', 'f']

data = "6lauqIqaJelwkFKM".encode()
hex_str = ''

for i in range(len(data) - 1, -1, -1):
	hex_str += table[(data[i] & 0xf0) >> 4]
	hex_str += table[data[i] & 0xf]

rsa_m = int(hex_str, 16)

rsa_c = rsa.core.encrypt_int(rsa_m, rsa_e, rsa_n)

print(hex(rsa_c))
```

这次得到的结果与请求参数的一致，再次发送请求已经可以获取到数据了。

![avatar](https://s2.ax1x.com/2019/06/04/VYLrZQ.png)

（看了文档才知道白折腾了。。。文档里面有说明是逆序的）
![avatar](https://s2.ax1x.com/2019/06/04/VYxdHO.png)

经过上面的分析可以大概得出请求的流程：前端先把请求的参数包装成json字符串，使用一个固定的密钥加密得到第一次加密的数据，再使用随机生成的密钥加密第一次加密生成的数据最终得到传递给服务器的参数，接着加密随机生成的密钥传递给服务器。服务器端用私钥解密得到前端随机生成的密钥，再用这个密钥对加密过的参数解密，经过两次解密最终得到参数。

关于csrf_token参数，这个是用户登录以后才会有的，之后可能会补充更新一下这部分的接口，另外如果用postman之类的工具请求接口好像直接把参数放在query部分才行，一开始放到post body获取不到数据。

最后把获取评论的接口贴一下：

url: https://music.163.com/weapi/v1/resource/comments/ID
method: POST
query: csrf_token
params: ID(string), offset(int), limit(int), total(boolean), csrf_token(string)

当请求歌曲评论时，ID的格式是R_SO_4_ + 歌曲ID。
歌单：A_PL_0 + 歌单ID。
电台：A_DJ_1 + 电台ID

其他类型的评论ID可以在开发者模式下获取相应请求url得到。

### 2.获取歌词 ###

通过上面的分析已经得到参数加密的方法，所以我们只需要知道请求这个接口都有什么参数就可以了。

example:

> {"id":"1361988914","lv":-1,"tv":-1,"csrf_token":""}

lv,tv这几个参数暂时不知道作用是什么，有兴趣的可以自己研究研究。

url: https://music.163.com/weapi/song/lyric
method: POST
query: csrf_token
params: ID(int), lv(int), lv(int), csrf_token(string)

### 3.获取播放列表 ###

在网页打开播放列表得到的列表详情是在后端处理过直接返回给浏览器的，没有暴露接口，但是通过官方提供的iframe插件可以得到一个接口，例如这个歌单：https://music.163.com/outchain/player?type=0&id=2821194580&auto=1&height=430 ，加密方式跟上文一样：

example:

>{"id":"2821194580","ids":"[\"2821194580\"]","limit":10000,"offset":0,"csrf_token":""}

id参数是必须的，ids参数的作用暂时不知道是什么。但是这个接口目前好像没办法正常使用，limit和offset设置是无效的，一次最多获取1000条歌曲数据，而且也无法获取订阅状态，还需要再研究研究。

url: https://music.163.com/weapi/playlist/detail
method: POST
params: id(int), ids(array), limit(int), offset(int), csrf_token(string)

### 4.搜索 ###

#### 4.1 推荐 ####

example：

> {"s":"陈奕迅","limit":"8","csrf_token":""}

url: https://music.163.com/weapi/cloudsearch/get/web
method: POST
query: csrf_token
params: s(string), limit(int), csrf_token(string)

#### 4.2 各种搜索 ####

example:

> {"hlpretag":"<span class=\"s-fc7\">","hlposttag":"</span>","s":"陈奕迅","type":"1","offset":"0","total":"true","limit":"30","csrf_token":""}

前面hlpretag和hlposttag暂时不知道作用是什么，可以不加进去，如果要加进去加密的时候记得要把转义符号也当成字符串的一部分，在python中可以在字符串前面加r表示raw string。
其中type是搜索的类型：
1 歌曲
10 专辑
100 歌手
1000 歌单
1002 用户
1006 歌词
1009 电台
1014 视频
其他的参数比较明显就不细说了。

url: https://music.163.com/weapi/cloudsearch/get/web
method: POST
query: csrf_token
params: s(string), type(int), offset(int), total(int), limit(int), csrf_token(string)

### 5.用户Token ###

先分析怎么通过手机登录获取token。
这个稍微复杂点，因为传递给服务器的参数中password是经过加密的，只要找出加密的方法就可以了。首先还是找到跟加密password相关的函数，这个断点多花点时间是可以找出来的。

在pt_frame_index.js这个文件里面发现如下代码：

```javascript
var i9b = {
    countrycode: gX3x.countrycode,
    phone:gX3x.mobile,
    password:k9b.lB4F(gX3x.password),
    rememberLogin:this.RU4Y.checked
}
```

其中lB4F这个函数就是加密password的函数，接着找到：

```javascript
p.lB4F = function(i9b) {
    return XC6w(LI2x(xC8u(i9b, !0), i9b.length * nA5F), !0)
};
```

```javascript
var nA5F = 8;

var xC8u = function() {
    var GB0x = function(i) {
        return i % 32
    },
    Gx0x = function(i) {
        return 32 - nA5F - i % 32
    };
    return function(cQ2x, Gw0x) {
        var Xr6l = [],
        lI4M = (1 << nA5F) - 1,
        mE5J = Gw0x ? GB0x: Gx0x;
        for (var i = 0,
        l = cQ2x.length * nA5F; i < l; i += nA5F) Xr6l[i >> 5] |= (cQ2x.charCodeAt(i / nA5F) & lI4M) << mE5J(i);
        return Xr6l
    }
} ();
```

```javascript
var LI2x = function(x, y) {
    x[y >> 5] |= 128 << y % 32;
    x[(y + 64 >>> 9 << 4) + 14] = y;
    var a = 1732584193,
    b = -271733879,
    c = -1732584194,
    d = 271733878;
    for (var i = 0,
    l = x.length,
    bFW5b, bFX5c, bGc5h, bGd5i; i < l; i += 16) {
        bFW5b = a;
        bFX5c = b;
        bGc5h = c;
        bGd5i = d;
        a = qb5g(a, b, c, d, x[i + 0], 7, -680876936);
        d = qb5g(d, a, b, c, x[i + 1], 12, -389564586);
        c = qb5g(c, d, a, b, x[i + 2], 17, 606105819);
        b = qb5g(b, c, d, a, x[i + 3], 22, -1044525330);
        a = qb5g(a, b, c, d, x[i + 4], 7, -176418897);
        d = qb5g(d, a, b, c, x[i + 5], 12, 1200080426);
        c = qb5g(c, d, a, b, x[i + 6], 17, -1473231341);
        b = qb5g(b, c, d, a, x[i + 7], 22, -45705983);
        a = qb5g(a, b, c, d, x[i + 8], 7, 1770035416);
        d = qb5g(d, a, b, c, x[i + 9], 12, -1958414417);
        c = qb5g(c, d, a, b, x[i + 10], 17, -42063);
        b = qb5g(b, c, d, a, x[i + 11], 22, -1990404162);
        a = qb5g(a, b, c, d, x[i + 12], 7, 1804603682);
        d = qb5g(d, a, b, c, x[i + 13], 12, -40341101);
        c = qb5g(c, d, a, b, x[i + 14], 17, -1502002290);
        b = qb5g(b, c, d, a, x[i + 15], 22, 1236535329);
        a = pT5Y(a, b, c, d, x[i + 1], 5, -165796510);
        d = pT5Y(d, a, b, c, x[i + 6], 9, -1069501632);
        c = pT5Y(c, d, a, b, x[i + 11], 14, 643717713);
        b = pT5Y(b, c, d, a, x[i + 0], 20, -373897302);
        a = pT5Y(a, b, c, d, x[i + 5], 5, -701558691);
        d = pT5Y(d, a, b, c, x[i + 10], 9, 38016083);
        c = pT5Y(c, d, a, b, x[i + 15], 14, -660478335);
        b = pT5Y(b, c, d, a, x[i + 4], 20, -405537848);
        a = pT5Y(a, b, c, d, x[i + 9], 5, 568446438);
        d = pT5Y(d, a, b, c, x[i + 14], 9, -1019803690);
        c = pT5Y(c, d, a, b, x[i + 3], 14, -187363961);
        b = pT5Y(b, c, d, a, x[i + 8], 20, 1163531501);
        a = pT5Y(a, b, c, d, x[i + 13], 5, -1444681467);
        d = pT5Y(d, a, b, c, x[i + 2], 9, -51403784);
        c = pT5Y(c, d, a, b, x[i + 7], 14, 1735328473);
        b = pT5Y(b, c, d, a, x[i + 12], 20, -1926607734);
        a = pQ5V(a, b, c, d, x[i + 5], 4, -378558);
        d = pQ5V(d, a, b, c, x[i + 8], 11, -2022574463);
        c = pQ5V(c, d, a, b, x[i + 11], 16, 1839030562);
        b = pQ5V(b, c, d, a, x[i + 14], 23, -35309556);
        a = pQ5V(a, b, c, d, x[i + 1], 4, -1530992060);
        d = pQ5V(d, a, b, c, x[i + 4], 11, 1272893353);
        c = pQ5V(c, d, a, b, x[i + 7], 16, -155497632);
        b = pQ5V(b, c, d, a, x[i + 10], 23, -1094730640);
        a = pQ5V(a, b, c, d, x[i + 13], 4, 681279174);
        d = pQ5V(d, a, b, c, x[i + 0], 11, -358537222);
        c = pQ5V(c, d, a, b, x[i + 3], 16, -722521979);
        b = pQ5V(b, c, d, a, x[i + 6], 23, 76029189);
        a = pQ5V(a, b, c, d, x[i + 9], 4, -640364487);
        d = pQ5V(d, a, b, c, x[i + 12], 11, -421815835);
        c = pQ5V(c, d, a, b, x[i + 15], 16, 530742520);
        b = pQ5V(b, c, d, a, x[i + 2], 23, -995338651);
        a = pA5F(a, b, c, d, x[i + 0], 6, -198630844);
        d = pA5F(d, a, b, c, x[i + 7], 10, 1126891415);
        c = pA5F(c, d, a, b, x[i + 14], 15, -1416354905);
        b = pA5F(b, c, d, a, x[i + 5], 21, -57434055);
        a = pA5F(a, b, c, d, x[i + 12], 6, 1700485571);
        d = pA5F(d, a, b, c, x[i + 3], 10, -1894986606);
        c = pA5F(c, d, a, b, x[i + 10], 15, -1051523);
        b = pA5F(b, c, d, a, x[i + 1], 21, -2054922799);
        a = pA5F(a, b, c, d, x[i + 8], 6, 1873313359);
        d = pA5F(d, a, b, c, x[i + 15], 10, -30611744);
        c = pA5F(c, d, a, b, x[i + 6], 15, -1560198380);
        b = pA5F(b, c, d, a, x[i + 13], 21, 1309151649);
        a = pA5F(a, b, c, d, x[i + 4], 6, -145523070);
        d = pA5F(d, a, b, c, x[i + 11], 10, -1120210379);
        c = pA5F(c, d, a, b, x[i + 2], 15, 718787259);
        b = pA5F(b, c, d, a, x[i + 9], 21, -343485551);
        a = mG5L(a, bFW5b);
        b = mG5L(b, bFX5c);
        c = mG5L(c, bGc5h);
        d = mG5L(d, bGd5i)
    }
    return [a, b, c, d]
};
```

```javascript
var mG5L = function(x, y) {
    var bFc4g = (x & 65535) + (y & 65535),
    cwl5q = (x >> 16) + (y >> 16) + (bFc4g >> 16);
    return cwl5q << 16 | bFc4g & 65535
};
var bem8e = function(q, a, b, x, s, t) {
    return mG5L(beu8m(mG5L(mG5L(a, q), mG5L(x, t)), s), b)
};
var qb5g = function(a, b, c, d, x, s, t) {
    return bem8e(b & c | ~b & d, a, b, x, s, t)
};
var pT5Y = function(a, b, c, d, x, s, t) {
    return bem8e(b & d | c & ~d, a, b, x, s, t)
};
var pQ5V = function(a, b, c, d, x, s, t) {
    return bem8e(b ^ c ^ d, a, b, x, s, t)
};
var pA5F = function(a, b, c, d, x, s, t) {
    return bem8e(c ^ (b | ~d), a, b, x, s, t)
};
```

```javascript
var XC6w = function() {
    var bFG5L = "0123456789abcdef",
    GB0x = function(i) {
        return i % 4
    },
    Gx0x = function(i) {
        return 3 - i % 4
    };
    return function(iL3x, Gw0x) {
        var bu0x = [],
        mE5J = Gw0x ? GB0x: Gx0x;
        for (var i = 0,
        l = iL3x.length * 4; i < l; i++) {
            bu0x.push(bFG5L.charAt(iL3x[i >> 2] >> mE5J(i) * 8 + 4 & 15) + bFG5L.charAt(iL3x[i >> 2] >> mE5J(i) * 8 & 15))
        }
        return bu0x.join("")
    }
} ();
```

看起来眼花缭乱，但是我们并不需要知道各个函数的作用，直接用python的语法转换一下。


