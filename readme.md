# UnionPay支付文档
------------------------------
##商户的申请
在银联网站的开放平台上申请成功之后，首先在银联的网站中下载相应的商户私钥证书，下载的证书均为5.1.0的版本。格式为.pfx。
###公共参数
目前所需要的公告参数，指的是从银联网站上获取的参数，如下
* 商户私钥证书.pfx
* 证书的ID，cerId，具体从银联提供的SDK中获取，目前利用的是银联提供的php版本的SDK中，通过读取商户私钥证书来获取相应的certId。注意！此处的certId和merId不是同一个东西，certId必须从.pfx中解析得到。目前可以在NodeJs版本的SDK中解析成功，利用```wopenssl```从.pfx中获取到的```serialNumber```就是未解析的```certId```，通过利用大整数方式(Big Integer)方式可以将其解析出来，即可获取响应的```certId```

###证书格式转换(.pfx --> .pem)
证书格式的转换通过SDK中提供的convert方法来执行，具体位置在根目录util文件下。
##银联证书
###私钥证书解析
在证书解析之前，需要将.pfx转化为.pem格式，在通过银联提供的SDK解析的时候，.pem文件里的加密部分是以```-----BEGIN PRIVATE KEY-----```开始，以```-----END PRIVATE KEY-----```结束。但是利用目前提供的iterm2命令行以及第三方工具(wopenssl)转换的.pem是以```-----BEGIN RSA PRIVATE KEY-----```开始，以```-----END RSA PRIVATE KEY-----```结束。其中加密部分也与银联SDK解析出来的部分不同，开始以为这两种不同的私钥通过同一种生成签名算法会生成不一样的签名，但是结果生成的签名一致，也许银联SDK和目前nodejs的SDK在解析.pfx时候，虽然利用的解析方法相同，这里的解析方法指的是pkcs12，但是解析程度不同，如果你知道，请告知一下，谢谢。解析完成之后，获取到privateKey。
##参数
所有参数的格式均为字符串，参数是以```x-www-form-urlencoded```形式发送至银联
具体需求参数如下
>version 版本号  (分为5.0.0以及5.1.0,目前均采用5.0.0)
>encoding 编码形式(只能小写)
>txnType 交易类型(默认采用01,其它方式请查询银联开发文档)
>txnSubType 交易子类型(默认采用01,其它方式请查询银联开发文档)
>bizType 业务类型(默认为000202)
>frontUrl 前台通知地址 (即交易成功之后跳转的商户地址)
>backUrl 后台通知地址
>signMethod 签名方法(默认为01)
>channelType 渠道类型(07-PC 08-手机)
>accessType 接入类型(默认为0)
>currencyCode 交易货币种类(境内商户固定为156，为人民币)
>merId 商户代码(请从银联开发文档中获取相关的ID)
>orderId 商户订单号(8-32位数字字母，不能含“-”或“_”,至少为8位，否则提交订单时会报错)
>txnTime 订单发送时间 (并不是订单在商城中创建的时间，应是目前发送至银联时的当前时间)
>txnAmt 交易金额，单位为分
>payTimeout 交易超时时间(超过超时时间调查询接口应答origRespCode不是A6或者00的就可以判断为失败)
>certId 证书ID(由于其解析算法还未实现，因此cerId仍然是从银联SDK中获取)
>signature 签名字符串(具体方式参见util目录下的buildParams签名生成部分)

当所有的参数发送至目标url之后，会跳转至银联的支付页面，之后完成支付


###时间形式
要发送过去的时间格式必须为```20180302165718```，这样的YYYYMMDDHHmmss形式。禁止其它格式的时间发送。
###发送至远端
调用SDK中的buildParams方式，这种方式接受orderId，txnAmt，channelType, frontUrl这几个参数，此方法会返回一个json对象，里面包含所有必要信息，只需要将json对象通过表单的形式推送至银联的支付端口，即可生成相应的订单以及收款。然后再完成支付即可。
#UnionPay查询文档
##查询订单所需参数
调用SDK中query方法，传入的参数为orderId以及txnTime，其中会获取到银联端口传回来的数据，数据封装在了SDK中，首先将数据整理后，转换成用&连接的参数字符串，然后发送给银联相应端口，之后获取，解析其中是否支付成功，再进行签名验证步骤。

发送字符串过去时，银联返回回来的是json对象，验证回调签名的时候，返回的是Boolean值。

发送的参数具体如下
> version
> encoding
> signMethod
> txnType 固定为00，我也不知道为什么 问银联
> txnSubType 固定为00，我也不知道为什么 问银联
> bizType   固定为000000，我也不知道为什么 问银联
> accessType
> channelType
> orderId
> merId
> txnTime
> certId
> signature

在生产字符串的时候严格按照sort()排列方法来对key值进行排列。用&符号来连接，其中具体排列方法在SDK中的utilities中。

#Unionpay签名回调验证文档
通过获取银联返回过来的json对象，将签名字符串提取出来，然后通过```filter```方法对参数进行处理，将一些空的参数去除掉，之后生成用```&```符号连接的参数字符串。然后进行签名的生成，调用```crypto```中的```verify```方式和银联的公钥进行匹配，算法均为```sha256```


#其余信息(证书配置，公共信息配置)
证书统一放置在根目录下的```certificates```下，如果有改动，请修改```/lib/unionpay.js```下的```privateKey```的配置目录，以及```/lib/verify.js```下的```publicKey```的配置目录。
其余信息，比如```merId```，```cerId```等配置在根目录的```config```文件夹中的```config.js```文件下。
目前仅仅完善了支付，查询，回调验证功能，以后还会完善退款功能

#对于根目录下的test.js作为测试流程用，安装包之后，直接
```
const Unionpay = require('unionpaysdk')
const payment = new Unionpay();
```
new一个新对象出来即可，剩下的请参照test.js流程