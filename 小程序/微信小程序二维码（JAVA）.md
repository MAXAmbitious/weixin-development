# 前言

最近在做小程序相关的项目，之前也在网上找过很多资料，遇到不少坑，和大家分享一下成果。
# 简介
小程序获取二维码有官网三种接口：
1. POST https://api.weixin.qq.com/cgi-bin/wxaapp/createwxaqrcode?access_token=ACCESS_TOKEN：获取小程序二维码，适用于需要的码数量较少的业务场景。
2. POST https://api.weixin.qq.com/wxa/getwxacode?access_token=ACCESS_TOKEN：获取小程序码，适用于需要的码数量较少的业务场景。
3. POST https://api.weixin.qq.com/wxa/getWXACodeUnlimit?access_token=ACCESS_TOKEN：获取小程序码，适用于需要的码数量极多的业务场景。
这三种方案前提都是要获取access_token，大家可根据各自需求选择。

# 获取access_token

核心代码如下：

```
private static AppletTokenResult getAppletToken(String key){
		AppletTokenResult tokenResult = new AppletTokenResult();
		String getaccess_tokenurl = "https://api.weixin.qq.com/cgi-bin/token?" +
				"grant_type=client_credential" +
				"&appid=" + APPID +
				"&secret=" + SECRET;
		RestTemplate restTpl = new RestTemplate();
		String tokenDataStr = restTpl.getForObject(getaccess_tokenurl, String.class);
		if(StringUtil.isEmpty(tokenDataStr)){
			throw new CommonException("小程序获取token失败！");
		}
		tokenResult = new Gson().fromJson(tokenDataStr, AppletTokenResult.class);
		if(StringUtil.isEmpty(tokenResult.getAccess_token())){
			throw new CommonException("小程序获取token失败！");
		}
		int expire = 2 * 60 * 55;
		int temp = tokenResult.getExpires_in();
		expire = temp == 0 ? expire : temp- 5 * 60;
		redisCache.put(key, (Serializable) tokenResult, expire);
		return tokenResult;
		
	}
```
AppletTokenResult类定义请求微信平台access_token返回信息：

```

import java.io.Serializable;

/**
 * 小程序获取token返回数据
 * @author 0200759
 *
 */
public class AppletTokenResult implements Serializable{

	/**
	 * 
	 */
	private static final long serialVersionUID = 2537928824643088525L;
	
	/**
	 * 小程序token
	 */
	private String access_token;
	
	/**
	 * token有效时间
	 */
	private int expires_in;
	
	public int getExpires_in() {
		return expires_in;
	}

	public void setExpires_in(int expires_in) {
		this.expires_in = expires_in;
	}

	public String getAccess_token() {
		return access_token;
	}

	public void setAccess_token(String access_token) {
		this.access_token = access_token;
	}

}

```
由于小程序token存在有效时间，超过这个时间需要重新从微信平台取，我这里把token放入redis缓存里并设置了缓存有效时间(比小程序token有效时间略小)。

# 获取二维码
代码如下：

```
/**
 * 小程序接口
 * @author 0200759
 *
 */
@Controller
@RequestMapping("/applet")
public class AppletController {
	
	
	/**
	 * @param res
	 * 调用二维码接口2(https://api.weixin.qq.com/wxa/getwxacodeunlimit?access_token=ACCESS_TOKEN)
	 */
	 @RequestMapping("/getQRCode")
	 public void getQRCode(HttpServletResponse res) {
	        RestTemplate rest = new RestTemplate();
	        InputStream inputStream = null;
	        try {
	        	AppletTokenResult appletTokenResult = CacheAppletUtil.get(Constants.APPLET_TOKEN);
	        	if(appletTokenResult == null || StringUtil.isEmpty(appletTokenResult.getAccess_token())){
	        		throw new CommonException("小程序token获取失败！");
	        	}
	            String url = "https://api.weixin.qq.com/wxa/getwxacodeunlimit?access_token="
	            			  +appletTokenResult.getAccess_token();
	            Map<String, Object> param= new HashMap<String, Object>();
	            param.put("scene", "xxxx");
	            param.put("path", "pages/index");
	            param.put("width", 430);
	            param.put("auto_color", false);
	            Map<String,Object> line_color = new HashMap<>();
	            line_color.put("r", 0);
	            line_color.put("g", 0);
	            line_color.put("b", 0);
	            param.put("line_color", line_color);
	            HttpHeaders headers = new HttpHeaders();
	            headers.setContentType(MediaType.APPLICATION_JSON);
	            HttpEntity<String> requestEntity = new HttpEntity<String>(new Gson().toJson(param), headers);
	            ResponseEntity<byte[]> entity = rest.exchange(url, HttpMethod.POST, 
	            		requestEntity, byte[].class, new Object[0]);
	            byte[] result = entity.getBody();
	            inputStream = new ByteArrayInputStream(result);
	            
	            IOUtils.copy(inputStream, res.getOutputStream());

	        } catch (Exception e) {
	            new CommonException("调用小程序生成微信永久小程序码URL接口异常!");
	        } finally {
	            if(inputStream != null){
	                try {
	                    inputStream.close();
	                } catch (IOException e) {
	                    e.printStackTrace();
	                }
	            }
	        }
	    }
}
```
这里使用的二维码方式是方案三getwxacodeunlimit，有以下几点需注意：
- param.put("scene", "xxxx")和param.put("path", "pages/index")必填
- 需要将参数json化，即new Gson().toJson(param)
- 我这里直接把返回的二维码字节流写到HttpServletResponse
-  不要在controller的getQRCode方法上@ResponseBody。。。

# 结语

从微信支付到微信小程序，一路坑过，希望我走过的路对你有点用。
