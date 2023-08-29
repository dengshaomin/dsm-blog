***
https://github.com/koush/AndroidAsync
***
本章节使用AndroidAsnyc部署一个vue项目在android本地

1.  使用vue-cli创建一个vue demo，并打包得到dist目录

!> 修改 vue.config.js增加publishPath防止白屏
```
const { defineConfig } = require('@vue/cli-service')
module.exports = defineConfig({
  transpileDependencies: true,
  lintOnSave: false,
  publicPath:'./',  //防止build后index.html打开空白
})
```

2.  将dist拷贝到android assets目录下

3.  AndroidMainifest.xml 增加权限

```
<uses-permission android:name="android.permission.INTERNET"></uses-permission>
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"></uses-permission>
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"></uses-permission>
```

4. demo 相关代码
```
class MainActivity : AppCompatActivity() {
    private lateinit var server: AsyncHttpServer
    private lateinit var web: WebView
    private lateinit var request: View

    @SuppressLint("MissingInflatedId")
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        web = findViewById(R.id.web)
        request = findViewById(R.id.request)
        server = AsyncHttpServer().apply {
//            directory(this@MainActivity, "/", "dist")
            listen(AsyncServer(), 8080)
        }
        Log.e("balance", getLocalIpAddress() ?: "")
        processHtml()
        processFile()
        web.settings.apply {
            javaScriptEnabled = true
            defaultTextEncodingName = "UTF-8"
        }
        web.webViewClient = WebViewClient()
//        web.loadUrl("file:///android_asset/dist/index.html");
        request.setOnClickListener {
//            var serverUrl = "file:///android_asset/dist/index.html"
            var serverUrl = "http://localhost:8080/"

            AsyncHttpClient.getDefaultInstance().executeString(
                AsyncHttpGet(serverUrl), object : AsyncHttpClient.StringCallback() {

                    override fun onCompleted(
                        e: Exception?,
                        source: AsyncHttpResponse?,
                        result: String?
                    ) {
                        if (result.isNullOrEmpty()) {
                            return
                        }
                        result?.let {
                            web.post {
//                                web.loadData(it, "text/html", "UTF-8")
                                web.loadDataWithBaseURL(serverUrl, it, "text/html", "UTF-8", null)
                            }
                        }
                    }
                })
//            web.loadUrl(serverUrl)
        }
    }

    private fun processHtml() {
        //处理html文件
        server["/", { request: AsyncHttpServerRequest, response: AsyncHttpServerResponse? ->
            val html = loadHTMLFromAssets("dist/index.html")
            response!!.setContentType("text/html")
            response!!.send(html)
        }]
    }

    private fun processFile() {
        //处理css、js文件
        server[".*", { request: AsyncHttpServerRequest, response: AsyncHttpServerResponse? ->
            // 获取文件路径
            val path = "dist${request.path}"
            try {
                val inputStream: InputStream = assets.open(path)
                response!!.sendStream(inputStream, inputStream.available().toLong())
            } catch (e: java.lang.Exception) {
                e.printStackTrace()
                response!!.end()
            }
        }]

    }

    private fun loadHTMLFromAssets(filename: String): String? {
        val assetManager = assets
        try {
            val inputStream: InputStream = assetManager.open(filename)
            val size: Int = inputStream.available()
            val buffer = ByteArray(size)
            inputStream.read(buffer)
            inputStream.close()
            return String(buffer)
        } catch (e: IOException) {
            e.printStackTrace()
        }
        return ""
    }

    private fun getLocalIpAddress(): String? {
        try {
            val networkInterfaces: Enumeration<NetworkInterface> =
                NetworkInterface.getNetworkInterfaces()
            while (networkInterfaces.hasMoreElements()) {
                val networkInterface: NetworkInterface = networkInterfaces.nextElement()
                val addresses: Enumeration<InetAddress> = networkInterface.inetAddresses
                while (addresses.hasMoreElements()) {
                    val address: InetAddress = addresses.nextElement()
                    if (!address.isLoopbackAddress && address.address.size === 4) {
                        return address.hostAddress
                    }
                }
            }
        } catch (e: java.lang.Exception) {
            e.printStackTrace()
        }
        return null
    }
}
```
5.  在pc和android端效果：

![](1.png ':size=60%')
![](0.png ':size=20%')

