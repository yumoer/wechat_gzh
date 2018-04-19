## 微信公众号开发第一天
* 验证服务器地址有效性
	* url 服务器的地址
	* token 用来加密签名，随意填写（尽量多、复杂）
* 将本地服务器地址变成外网跨域访问的网址
	* 内网穿透：将本地服务器端口号地址映射成外网
	* ngrok

* 验证服务器地址有效性
	* 将三个参数timestamp、nonce、token组合起来，进行字典序排序
	* 将排序完的字符串拼接在一起
	* 将拼接好字符串进行sha1加密
	* 将加密的字符串与微信传过来的签名对比，如果正确说明验证成功，

* 封装抽象代码
	* 将config配置文件封装起来config
	* 将中间件的回调函数封装起来confirm

* 获取access_token
	* 创建一个构造函数Wechat 用来处理access_token
	* 是在中间件中new出来的，所以函数体里面的内容会被执行
	* updateAccessToken 获取
		* 请求地址（需要appID和appsecret，通过this拿到，注意箭头函数问题），
		* 通过request-promise库（依赖于request）发送请求，返回一个promise对象，通过resolve方法返回请求结果
	* saveAccessToken 保存
		* 在获取access_token后，需要保存access_token
		* 保存方法放在utils/tools中，writeFileAsync
			* 注意data不能是js对象，需要转化
			* 以及./路径的问题（哪里最终调用代表哪个目录）此处引入path模块解决
		* 在updatetAccessToken下的then方法成功回调中调用此方法
	* 注意：我需要将调用的函数依次通过promise方式向下传递，所以每个方法必须return一个promise对象
* 获取expires_in
	* 在updateAccessToken中获取并设置凭据的过期时间
	* getAccessToken 读取不传值
		* 在获取expires_in后，需要保存expires_in
		* 读取方法readeFileAsync
			* 以及./路径的问题（哪里最终调用代表哪个目录）此处引入path模块解决 
		* 在fs传值，将buffer数据转成字符串
	* isValidAccessToken 检查有效性
		* 文件不存在/没有凭证/凭证过期 返回false
		* 当前时间小于凭证的过期时间  返回true（now < expires_in）
	* 先去本地读取凭据
		* 如果有
			* 检查凭据有没有过期
				* 如果没过期
					* 我们就直接使用
				* 如果过期了
					* 需要重新请求微信服务器的access_token，并保存在本地
        * 如果没有
	        * 需要重新请求微信服务器的access_token，并保存在本地
## 微信公众号开发第二天
* 注意promise不会在此去强调了，默认返回就是promise

* access_token真正的设计思路***
	* 真实开发时我们需要针对复杂的功能进行设计，需要构思设计思路
	* 根据access_token的特点，我们得出以下设计：
	* 先去本地读取凭据
		* 如果有
			* 检查凭据有没有过期
				* 如果没过期
					* 我们就直接使用
				* 如果过期了
					* 需要重新请求微信服务器的access_token，并保存在本地
        * 如果没有
	        * 需要重新请求微信服务器的access_token，并保存在本地
    * 有了设计思路，再为了复用、代码之间的联系等等特点，设计构造函数Wechat
    * 将功能需要使用的函数设计出来
    * 最后就是将文字翻译成代码~

* 封装Wechat构造函数，优化
	* 封装wechat文件
	* 优化少使用readFile方法 提前将凭据缓存在this上
	* 创建了一个this.fetchAccessToken()函数用来取得最终的access_token

* 获取用户消息
	* 微信服务器会发送两种类型消息给我们
		* get 验证服务器有效性
		* post 验证服务器有效性和发送用户消息
	* 用户发送过来的消息最终是xml数据，我们需要格式化处理一下，对应有三部。通过async await依次向下传递数据
		* 首先接受用户发来的流式数据，通过给req绑定data事件慢慢接收，在end事件返回数据
		* 将xml数据转化为js对象
			* 使用了xml2js的库进行转化
				* 我们可以通过npm网址来搜索我们想使用的包
				* 通过github来搜索我们开发的项目
		* 将js对象中的数据干掉

* 实现自动回复
	* 我们首先在confirm文件上实现了最简单的自动回复
	* 接下来封装了两个功能模块来使用最终的回复
		* reply 处理用户发送消息的事件类型，不同的事件对应不同的回应~
			* if else 逻辑嵌套较多，注意
			* 需要确认传入tpl中的options的msgType，注意
		* tpl 最终回复给用户的6种消息模板文件，消息类型是xml数据~
	* 注意事项：
		* 开发者在5秒内未回复任何内容 *
		* xml数据中Content为空 ***
		* 开发者回复了异常数据，比如JSON数据、xml数据中有多余空格等 *****
		* 出现以上三个问题，均会导致微信公众号报错：该公众号提供的服务出现故障，请稍后再试
	
	
* 上传素材接口
	* 一个函数做了两件事。
	* 临时素材
		* 一种接口
			* 上传4种素材类型
		* 请求url必须写type
	* 永久素材
		* 三种接口
			* news 没有type
				* 只有它需要通过请求体上传素材
			* 图片url 没有type
			* 其他素材类型 有type


## 微信公众号开发第三天
* 获取素材
	* 临时素材
		* get
		* 视频文件不支持https下载，调用该接口需http协议
		* access_token media_id
		* 返回值：
			* 视频素材 对象url
			* 非视频素材 数据以流的方式过来
	* 永久素材
		* post
			* access_token  请求体media_id
		* 返回值：
			* 图文素材 {}
			* 视频素材 {}
			* 其他类型素材 数据以流的方式过来
	* 注意：其他类型以流的方式发送的数据并不需要，所以直接resolve(url)，我们真正用的是news
	* 学会如何使用文档，如何构思功能

* 自定义菜单
	* create菜单
	* 坑，如果菜单之前创建过再次创建会报错（access_token是非法的）
		* 需要先删除之前的在创建
		* 菜单一般只创建一次就ok
	* 学会开发流程：
		* 先定义api接口
		* 在定义函数，实现主要功能
		* 再去调用函数，测试是否有效

* 微信公众号交互流程
	* 注意：开发者服务器一直都是与微信服务器做交互，并没有直接和用户对接

* JS-SDK
	* 设置JS接口安全域名 （必须将协议干掉！！！）
  	* 获取jsapi_ticket，临时票据。获取票据和凭据类似，它和凭据有类似的特点
	    - 请求接口次数极少，建议开发者缓存起来
	    - 7200秒刷新一次
	    - 完全复制凭据的方法重写一遍
		    - 需要修改write/readFile函数
		    - 注意json问题
  	* 签名算法
	  	* 获取4个值（注意大小写和名称不能错）
		  	* noncestr（随机字符串）
		  	* jsapi_ticket（票据）
		  	* timestamp（时间戳）
		  	* url（完整url）
	  	* 将其按URL键值对的格式组合起来
	  	* 参数按照字段名的ASCII码（字典序）从小到大排序，在通过&拼接在一起
	  	* 进行sha1加密得到最终签名
  	* 将signature,noncestr,timestamp三个参数渲染到页面上
  	
	* 页面中需要引入微信的js文件
	* 配置config（这里需要以字符串的形式，并且需要使用的接口必须填写）
	* 测试接口是否有权限


* 学习到的能力
	* 使用接口文档的能力
	* 解决错误的能力
	* 设计构思功能
	* 使用接口流程：
		* 先定义接口公共部分
		* 在定义使用接口的方法
			* 通过接口需要传入参数，需要仔细查阅接口文档
			* 目前微信公众号接口传入参数方式有以下几种：
				* GET 
					* 以查询字符串的形式传递参数数据
				* POST
					* 以查询字符串的形式传递参数数据
					* 以请求体(body)方式传递参数数据
					* 以formData方式传递媒体文件数据
						* 注意这种方式需要借用request库来使用
				* 注意：
					* 如果需要的参数仅仅是查询字符串的话，这种接口通常为GET请求方式
					* 一旦有需要相对保密的参数数据，接口都会用POST请求发送
			* 返回值！！！
		* 申明使用接口
				