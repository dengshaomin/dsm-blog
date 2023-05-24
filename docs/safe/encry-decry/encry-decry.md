

为了防止我们的数据泄露，我们往往会对数据进行加密，特别是敏感数据，我们要求的安全性更高。下面将介绍几种常用的加密算法使用。这些算法的加密对象都是基于二进制数据，如果要加密字符串就使用统一编码（如：utf8）进行编码后加密。

### 1、摘要算法（MD5,SHA1）（不可解密）

常用的摘要算法有MD5,SHA1。摘要算法是一个不可逆过程，就是无论多大数据，经过算法运算后都是生成固定长度的数据,一般结果使用16进制进行显示。

MD5和SHA1的区别：MD5结果是128位摘要，SHa1是160位摘要（可以设置一个额外的加密key）。那么MD5的速度更快，而SHA1的安全强度更高。

下面统一使用MD5算法进行说明，SHA1类似。

主要用途有：验证消息完整性，安全访问认证，数据签名。

*   消息完整性：由于每一份数据生成的MD5值不一样，因此发送数据时可以将数据和其MD5值一起发送，然后就可以用MD5验证数据是否丢失、修改。
*   安全访问认证：这是使用了算法的不可逆性质，（就是无法从MD5值中恢复原数据）对账号登陆的密码进行MD5运算然后保存，这样可以保证除了用户之外，即使数据库管理人员都无法得知用户的密码。
*   数字签名：这是结合非对称加密算法和CA证书的一种使用场景。

一般破解方法：字典法，就是将常用密码生成MD5值字典，然后反向查找达到破解目的，因此建议使用强密码。

**MD5的使用—对文件进行摘要。**

```
// 对文件进行MD5摘要
public static String getMD5(String path) {
 
    String pathName = path;
    String md5= "";
    try {
        File file = new File(pathName);
        FileInputStream ins = new FileInputStream(file);
        FileChannel ch = ins.getChannel();
        MappedByteBuffer byteBuffer = ch.map(FileChannel.MapMode.READ_ONLY, 0,file.length());       
        MessageDigest md = MessageDigest.getInstance("MD5");
        md.update(byteBuffer);
        ins.close();
        md5 = toHexString(md.digest());
    } catch (NoSuchAlgorithmException e) {
        e.printStackTrace();
    } catch (FileNotFoundException e) {
        e.printStackTrace();
    } catch (IOException e) {
        e.printStackTrace();
    }
    return md5;
}
 
// 以16进制编码进行输出
final static char hex[] = {'0','1','2','3','4','5','6','7','8','9','A','B','C','D','E','F'};
public static String toHexString(byte[] tmp){
    String s;
    char str[] = new char[tmp.length*2];
    int k =0;
    for (int i = 0; i < tmp.length; i++) {
        byte byte0 = tmp[i];
        str[k++] = hex[byte0>>>4&0xf];
        str[k++] = hex[byte0&0xf];
    }
    s=new String(str);
    return s;
}
```

**SHA1的使用**

```
//对文件进行SHA1摘要
public static String getSHA1(String path){
    String pathName = path;
    String sha1= "";
    try {
        File file = new File(pathName);
        FileInputStream ins = new FileInputStream(file);
        FileChannel ch = ins.getChannel();
        MappedByteBuffer byteBuffer = ch.map(FileChannel.MapMode.READ_ONLY, 0,file.length());       
        MessageDigest sha = MessageDigest.getInstance("SHA-1");
        sha.update(byteBuffer);
        ins.close();
        sha1 = toHexString(sha.digest());
    } catch (NoSuchAlgorithmException e) {
        e.printStackTrace();
    } catch (FileNotFoundException e) {
        e.printStackTrace();
    } catch (IOException e) {
        e.printStackTrace();
    }
    return sha1;
}
```

可以发现我们的关键代码就是

```
MessageDigest sha = MessageDigest.getInstance("SHA-1");
sha.update(byteBuffer);
ins.close();
byte[] r = sha.digest());
```

只是不同的算法初始化时不同罢了。MessageDigest.getInstance("SHA-1")
另外还可以使用

```
DigestUtils.sha1(data)；
DigestUtils.md5Hex(data)；
```

上面实现使用的是Apache下面的一个加解密开发包commons-codec

官方地址为：http://commons.apache.org/codec/
官方下载地址：http://commons.apache.org/codec/download_codec.cgi

### 2、对称加密算法（DES,AES）（可解密）

对称加密算法只是为了区分非对称加密算法。其中鲜明的特点是对称加密是加密解密使用相同的密钥，而非对称加密加密和解密时使用的密钥不一样。对于大部分情况我们都使用对称加密，而对称加密的密钥交换时使用非对称加密，这有效保护密钥的安全。非对称加密加密和解密密钥不同，那么它的安全性是无疑最高的，但是它加密解密的速度很慢，不适合对大数据加密。而对称加密加密速度快，因此混合使用最好。
常用的对称加密算法有：AES和DES.

*   DES：比较老的算法，一共有三个参数入口（原文，密钥，加密模式）。而3DES只是DES的一种模式，是以DES为基础更安全的变形，对数据进行了三次加密，也是被指定为AES的过渡算法。

*   AES:高级加密标准，新一代标准，加密速度更快，安全性更高（不用说优先选择）

#### DES的使用

和AES类似，指定为DES就行。3DES指定为”DESede”,DES密钥长度是56位，3DES加长了密钥长度，可以为112位或168位，所以安全性提高，速度降低。工作模式和填充模式标准和AES一样

```
/**
 * 使用DES对字符串加密
 * @param str utf8编码的字符串
 * @param key 密钥（56位，7字节）
 * @return 加密结果
 * @throws Exception
 */
public static byte[] desEncrypt(String str, String key) throws Exception { 
   if (str == null || key == null) return null; 
   Cipher cipher = Cipher.getInstance("DES/ECB/PKCS5Padding"); 
   cipher.init(Cipher.ENCRYPT_MODE, new SecretKeySpec(key.getBytes("utf-8"), "DES")); 
   byte[] bytes = cipher.doFinal(str.getBytes("utf-8")); 
   return  bytes;
} 
    
/**
 * 使用DES对数据解密
 * @param bytes utf8编码的二进制数据
 * @param key 密钥（16字节）
 * @return 解密结果
 * @throws Exception
 */
public static String desDecrypt(byte[] bytes, String key) throws Exception { 
   if (bytes == null || key == null) return null; 
   Cipher cipher = Cipher.getInstance("DES/ECB/PKCS5Padding"); 
   cipher.init(Cipher.DECRYPT_MODE, new SecretKeySpec(key.getBytes("utf-8"), "DES")); 
   bytes = cipher.doFinal(bytes);
   return new String(bytes, "utf-8"); 
}
```

#### AES的使用

AES密钥长度可以选择128位【16字节】，192位【24字节】和256位【32字节】密钥（其他不行，因此别乱设密码哦）。

```
/**使用AES对字符串加密
 * @param str utf8编码的字符串
 * @param key 密钥（16字节）
 * @return 加密结果
 * @throws Exception
 */
public static byte[] aesEncrypt(String str, String key) throws Exception { 
   if (str == null || key == null) return null; 
   Cipher cipher = Cipher.getInstance("AES/ECB/PKCS5Padding"); 
   cipher.init(Cipher.ENCRYPT_MODE, new SecretKeySpec(key.getBytes("utf-8"), "AES")); 
   byte[] bytes = cipher.doFinal(str.getBytes("utf-8")); 
   return  bytes;
} 
/**使用AES对数据解密
 * @param bytes utf8编码的二进制数据
 * @param key 密钥（16字节）
 * @return 解密结果
 * @throws Exception
 */
public static String aesDecrypt(byte[] bytes, String key) throws Exception { 
   if (bytes == null || key == null) return null; 
   Cipher cipher = Cipher.getInstance("AES/ECB/PKCS5Padding"); 
   cipher.init(Cipher.DECRYPT_MODE, new SecretKeySpec(key.getBytes("utf-8"), "AES")); 
   bytes = cipher.doFinal(bytes);
   return new String(bytes, "utf-8"); 
}  
```

上面代码是对字符串进行的加解密。但要注意的是AES算法的所有参数都是字节码的（包括密钥）。因此字符串字符需要转换成字节码后进行加密str.getBytes("utf-8")按照字符串的编码进行转换。另外参数：”AES/ECB/PKCS5Padding”在加密和解密时必须相同，可以直接写”AES”,这样就是使用默认模式。分别的意思为：AES是加密算法，ECB是工作模式，PKCS5Padding是填充方式。

AES是分组加密算法，也称块加密。每一组16字节。这样明文就会分成多块。当有一块不足16字节时就会进行填充。

一共有四种工作模式：

*   ECB 电子密码本模式：相同的明文块产生相同的密文块，容易并行运算，但也可能对明文进行攻击。
*   CBC 加密分组链接模式：一块明文加密后和上一块密文进行链接，不利于并行，但安全性比ECB好，是SSL,IPSec的标准。
*   CFB 加密反馈模式：将上一次密文与密钥运算，再加密。隐藏明文模式，不利于并行，误差传递。
*   OFB 输出反馈模式：将上一次处理过的密钥与密钥运算，再加密。隐藏明文模式，不利于并行，有可能明文攻击，误差传递。

PKCS5Padding的填充方式是差多少字节就填数字多少；刚好每一不足16字节时，那么就会加一组填充为16.还有其他填充模式【Nopadding,ISO10126Padding】（不影响算法，加密解密时一致就行）。


### 3、非对称加密（RSA）（可解密）

这里主要对RSA进行介绍。
对称加密加密解密使用的是相同的密钥，而非对称加密加密解密时使用的不同的密钥，分为公钥（public key）和私钥(private key).公钥可以公开，而私钥自己保存。它利用的是两个大质数相乘十分容易，而对其乘积进行因素分解十分困难。这样就可以将乘积作为密钥了,这个乘积为N值，根据两个大质数选择e和生成d,删掉两个大质数。这样（N,e）为公钥，（N,d）为私钥，公钥无法破解出私钥（不作详细介绍，我们不是研究算法的）。由于非对称加密的密钥生成麻烦，所以无法做到一次一密，而且其加密速度很慢，无法对大量数据加密。因此最常用的使用场景就是数字签名和密码传输，用作数字签名时使用私钥加密，公钥解密；用作加密解密时，使用公钥加密，私钥解密。

需要注意的是RSA加密是有长度限制的，1024位密钥可以加密128字节（1024位），不满128字节的使用随机数填充，但是RSA实现中必须要加随机数（11字节以上），所以明文长度最大为117字节，然后剩下的加入随机数。这也产生了每次加密结果每一次都不一样的特点。

如果明文长度超过限制怎么办？

*   1.可以与对称加密混合使用，一次一密产生对称加密的密钥，然后使用此密钥进行数据对称加密，再使用RSA私钥对对称密钥加密，一起保存。解密时使用公钥解密出密钥，然后进行数据解密。
*   2.可以分段加密。将明文按117字节分成多段，加密后再拼接起来。由于每一段密文长度都是128字节，所以解密时按照128字节分段解密。

####    Java的RSA密钥生成与使用

####    简单使用
下面是Java中的使用方法，先是生成密钥对，然后加密，再解密。需要注意的是这个方法是不能跨语言使用的，因为里面对公钥和私钥用到的序列化是java的序列化。
由于加密后的密文都是字节码形式的，我们要以字符串方式保存或传输的话，可以使用Base64编码。
```
public class RSAUtil {
    /** 指定加密算法为RSA */
    private static String ALGORITHM = "RSA";
    /*指定加密模式和填充方式*/
    private static String ALGORITHM_MODEL = "RSA/ECB/PKCS1Padding";
    /** 指定key的大小，一般为1024，越大安全性越高 */
    private static int KEYSIZE = 1024;
    /** 指定公钥存放文件 */
    private static String PUBLIC_KEY_FILE = "PublicKey";
    /** 指定私钥存放文件 */
    private static String PRIVATE_KEY_FILE = "PrivateKey";
 
    /**
     * 生成密钥对
     */
    private static void generateKeyPair() throws Exception {
        /** RSA算法要求有一个可信任的随机数源 */
        SecureRandom sr = new SecureRandom();
        /** 为RSA算法创建一个KeyPairGenerator对象 */
        KeyPairGenerator kpg = KeyPairGenerator.getInstance(ALGORITHM);
        /** 利用上面的随机数据源初始化这个KeyPairGenerator对象 */
        kpg.initialize(KEYSIZE, sr);
        /** 生成密匙对 */
        KeyPair kp = kpg.generateKeyPair();
        /** 得到公钥 */
        Key publicKey = kp.getPublic();
        /** 得到私钥 */
        Key privateKey = kp.getPrivate();
        /** 用对象流将生成的密钥写入文件 */
        ObjectOutputStream oos1 = new ObjectOutputStream(new FileOutputStream(
                PUBLIC_KEY_FILE));
        ObjectOutputStream oos2 = new ObjectOutputStream(new FileOutputStream(
                PRIVATE_KEY_FILE));
        oos1.writeObject(publicKey);
        oos2.writeObject(privateKey);
        /** 清空缓存，关闭文件输出流 */
        oos1.close();
        oos2.close();
    }
 
    /**
     * 加密方法 source： 源数据
     */
    public static byte[] encrypt(String source) throws Exception {
 
        /** 将文件中的公钥对象读出 */
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(
                PUBLIC_KEY_FILE));
        Key key = (Key) ois.readObject();
        ois.close();
        /** 得到Cipher对象来实现对源数据的RSA加密 */
        Cipher cipher = Cipher.getInstance(ALGORITHM_MODEL);
        cipher.init(Cipher.ENCRYPT_MODE, key);
        byte[] b = source.getBytes();
        /** 执行加密操作 */
        byte[] b1 = cipher.doFinal(b);
        return b1;
    }
 
    /**
     * 解密算法 cryptograph:密文
     */
    public static String decrypt(byte[] cryptograph) throws Exception {
        /** 将文件中的私钥对象读出 */
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(
                PRIVATE_KEY_FILE));
        Key key = (Key) ois.readObject();
        /** 得到Cipher对象对已用公钥加密的数据进行RSA解密 */
        Cipher cipher = Cipher.getInstance(ALGORITHM_MODEL);
        cipher.init(Cipher.DECRYPT_MODE, key);
        /** 执行解密操作 */
        byte[] b = cipher.doFinal(cryptograph);
        return new String(b);
    }
 
    public static void main(String[] args) throws Exception {
        generateKeyPair();//生成密钥对
        String source = "Hello World!";// 要加密的字符串
        byte[] cryptograph = encrypt(source);// 生成的密文
 
        //可以将密文进行base64编码进行传输
        System.out.println(new String(Base64.encode(cryptograph)));
 
        String target = decrypt(cryptograph);// 解密密文
        System.out.println(target);
    }
}
```

####    RSA密钥使用Base64编码

要灵活使用肯定不能使用java的序列化保存了，我们对上面的generateKeyPair()方法进行改写。通过密钥生成器生成公钥，私钥后，调用publicKey.getEncoded()和privateKey.getEncoded()，此时它生成的比特编码是有独特格式的（公钥是X.509，私钥是PKCS#8）可以使用publicKey.getFormat()，privateKey.getFormat();进行查看。之后对字节码进行Base64编码就行了。

密钥生成方法

```
//以base64编码密钥
public Map<String ,String> generateKeyPair1() throws Exception{
    SecureRandom sr = new SecureRandom();
    KeyPairGenerator kpg = KeyPairGenerator.getInstance("RSA");
    kpg.initialize(1024, sr);
    KeyPair kp = kpg.generateKeyPair();
    Key publicKey = kp.getPublic();
    Key privateKey = kp.getPrivate();
    byte[] pb = publicKey.getEncoded();
    String pbStr =  new String(Base64.encode(pb));
    byte[] pr = privateKey.getEncoded();
    String prStr =  new String(Base64.encode(pr));
    Map<String, String> map = new HashMap<String, String>();
    map.put("publicKey",pbStr);
    map.put("privateKey", prStr);
    return map;
}
```

恢复密钥方法，使用各自不同的编码形式恢复

```
//从base64编码的公钥恢复公钥
public PublicKey getPulbickey(String key_base64) throws Exception{
    byte[] pb = Base64.decode(key_base64).getBytes();
    X509EncodedKeySpec keySpec = new X509EncodedKeySpec(pb);
    KeyFactory  keyfactory = KeyFactory.getInstance("RSA");
    return keyfactory.generatePublic(keySpec);
}
 
//从base64编码的私钥恢复私钥
public PrivateKey getPrivatekey(String key_base64) throws Exception{
    byte[] pb = Base64.decode(key_base64).getBytes();
    PKCS8EncodedKeySpec keySpec = new PKCS8EncodedKeySpec(pb);
    KeyFactory  keyfactory = KeyFactory.getInstance("RSA");
    return keyfactory.generatePrivate(keySpec);
}
```

加密解密方法都类似下面，PrivateKey和PublicKey是Key的子接口。

```
/** 执行加密操作 */
public static byte[] encrypt(Key key,byte[] source) throws Exception{
    Cipher cipher = Cipher.getInstance("RSA/ECB/PKCS1Padding");
    cipher.init(Cipher.ENCRYPT_MODE, key);
    byte[] ciphertext = cipher.doFinal(source);
    return ciphertext;
}
 
/** 执行加密操作 */
public static byte[] decrypt(Key key,byte[] ciphertext) throws Exception{
    Cipher cipher = Cipher.getInstance("RSA/ECB/PKCS1Padding");
    cipher.init(Cipher.DECRYPT_MODE, key);
    byte[] source = cipher.doFinal(ciphertext);
    return source;
}
```

####    记录RSA的密钥特征值并进行密码恢复

所谓特征值就是RSA中公钥（N,e）私钥（N,d）的三个值：N,e,d。只要有这三个值我们就可以恢复密钥了。这是实际开发中常用的方法。首先是提取特征值，我们需要将PublicKey强制转换为RSAPublicKey.然后获取，看代码。

```
//提取特征值保存，以base64编码密钥
public static Map<String ,String> generateKeyPair2() throws Exception {
    SecureRandom sr = new SecureRandom();
    KeyPairGenerator kpg = KeyPairGenerator.getInstance("RSA");
    kpg.initialize(1024, sr);
    KeyPair kp = kpg.generateKeyPair();
    Key publicKey = kp.getPublic();
    Key privateKey = kp.getPrivate();
    RSAPublicKey rpk = (RSAPublicKey)publicKey;
    RSAPrivateKey rpr= (RSAPrivateKey)privateKey;
    //三个特征值都是BigInteger类型。
    BigInteger N = rpk.getModulus();//N值
    BigInteger e = rpk.getPublicExponent();//e值
    BigInteger d  = rpr.getPrivateExponent();//d值
    Map<String, String> map = new HashMap<String, String>();
    //将BigInteger转为byte[],然后以base64保存
    map.put("N",new String(Base64.decode(N.toByteArray())));
    map.put("e", new String(Base64.decode(e.toByteArray())));
    map.put("d", new String(Base64.decode(d.toByteArray())));
    return map;
}
```

利用三个特征值就可以非常容易恢复密钥了。

```
//从base64编码的特征值(N,e)恢复公钥
public static PublicKey getPulbickey(String N_Str,String e_Str) throws Exception{
    BigInteger N = new BigInteger(1, Base64.decode(N_Str.getBytes()));
    BigInteger e = new BigInteger(1, Base64.decode(e_Str.getBytes()));
    KeyFactory kf = KeyFactory.getInstance("RSA");
    RSAPublicKeySpec ps = new RSAPublicKeySpec(N, e);
    PublicKey pkey = kf.generatePublic(ps);
    return pkey;
}
 
//从base64编码的特征值（N,d）恢复私钥
public static PrivateKey getPrivatekey(String N_Str,String d_Str) throws Exception{
    BigInteger N = new BigInteger(1, Base64.decode(N_Str.getBytes()));
    BigInteger d = new BigInteger(1, Base64.decode(d_Str.getBytes()));
    KeyFactory kf = KeyFactory.getInstance("RSA");
    RSAPrivateKeySpec ps = new RSAPrivateKeySpec(N, d);
    PrivateKey pkey = kf.generatePrivate(ps);
    return pkey;
}
```

### 4、编码的使用
常见的编码有Base64,HEX和对URL的编码。这都是为了实际需要才进行的编码。HEX是编码成16进制字符，MD5一般就是以HEX进行编码，这不说了。

####    Base64

Base64一开始是为了解决邮件中不能传文件和图片问题而使用的，将无法阅读的二进制码转化成字符形式，字符为（A-Za-z0-9+/）。它的原理是将3个8位字节（24位）转化为4个6位字节（24位），之后在6位的前面补两个0，形成8位一个字节形式，如果剩下的不足3字节，则用0填充，输出字符使用”=”,所以编码后文本可能出现1个或2个’=’.这样就将原本3个字节变成了4个字节，那就是64种编码了。当然，除了对二进制数据编码，还可以对字符串编码来隐藏明文，让别人不那么容易看懂。

由于jdk中的base64是不开发使用了 ，所有需要下载到网上下载Base64包，我使用的是 javaBase64-1.2.jar，另外android sdk中是带有base64的，位置是android.util.Base64

```
/*这里使用的是android.util.base64*/
byte[] input = "hello world".getBytes("utf8");
//编码    
byte[] encodeData = Base64.encode(input , 0);
//解码        
byte[] result = Base64.decode(encodeData , 0);
```
####    URL的编码

url一般使用的都是英文、数字和某些符号，而对于特殊符号，中文等这些是不允许使用的。因此我们要在url请求中加入特殊符号，中文等就需要对它们进行编码。http请求时，url部分是必须编码的，get的请求字段可以不进行url编码。比如

http://www.baidu.com/中文?wd=国际

“中文”必须进行url编码，“国际”可以不用。

那url编码到底是怎么进行编码的呢？

都是在16进制前面加上‘%’表示。对于一些字符使用的是“%xx”，而对于中文，就是多个“%xx%xx%xx”，xx的数字有编码的16进制决定（没有指定字符编码（utf8），则使用默认编码），然后每一字节前面加”%”。

Android 中提供的URL编码解码方法。

```
String d = URLEncoder.encode('中文'，"utf8");
String f = URLDecoder.decode("%20");
```

