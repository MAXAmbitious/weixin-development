该API用于获取预授权码。预授权码用于公众号或小程序授权时的第三方平台方安全验证。
web层代码如下:

```
/**
	 * 授权回调
	 * @param resp
	 */
	@RequestMapping(value = "/wechat/redirectAuthPage")
	@ResponseBody
	public void redirectAuthPage(HttpServletRequest request,HttpServletResponse resp) throws IOException {
		resp.sendRedirect(WxThirdPartyAuthUtil.getRedirectAuthUrl(request));
	}
```
WxThirdPartyAuthUtil类相关代码：

```
/**
	 * 获取第三方回调授权地址
	 * @param request 
	 * @return
	 */
	public static String getRedirectAuthUrl(HttpServletRequest request){
		String url="";
		try {
			url = "https://mp.weixin.qq.com/cgi-bin/componentloginpage?"
					+ "component_appid=%s&pre_auth_code=%s&redirect_uri=%s";
			url = String.format(url, component_appid,WxThirdPartyAuthUtil.getPreAuthCode(),
					URLEncoder.encode(UrlUtil.getDomainWithContext(request)+"/wechat/$APPID$/callback", "utf-8"));
			LOGGER.error("第三方回调授权地址"+url);
		} catch (UnsupportedEncodingException e) {
			LOGGER.error(e.toString());
		}
		return url;
	}
	
/**
	 * 获取预授权码
	 * @return
	 */
	public static String getPreAuthCode() {
		String preAuthCode ="";
		if(redisCache.hasKey(WxThirdPartyAuthConstants.PRE_AUTH_CODE)){
			preAuthCode = (String) redisCache.get(WxThirdPartyAuthConstants.PRE_AUTH_CODE);
			/** 判断缓存中是否存在pre_auth_code */
			if (StringUtils.isNotEmpty(preAuthCode)) {
				/** 如果存在，直接返回pre_auth_code的值 */
				return preAuthCode;
			}
		}
		/** token若不存在，重新请求token */
		String componentAccessToken = getComponentAccessToken();
		LOGGER.info("ThirdPartyServiceImpl:getPreAuthCode:componentAccessToken={}", componentAccessToken);
		/** 替换Url中的{component_access_token} */
		String url = "https://api.weixin.qq.com/cgi-bin/component/api_create_preauthcode?"
				+ "component_access_token={component_access_token}"
				.replace("{component_access_token}", componentAccessToken);
		JSONObject json = new JSONObject();
		json.put("component_appid", component_appid);
		/** 发送Https请求到微信 */
		JSONObject resultJson;
		try {
			resultJson = HttpClientUtil.postJson(url, json);
			/** 在返回结果中获取pre_auth_code */
			preAuthCode = resultJson.getString(WxThirdPartyAuthConstants.PRE_AUTH_CODE);
			/** 保存pre_auth_code，并设置有效时间 */
			redisCache.put(WxThirdPartyAuthConstants.PRE_AUTH_CODE, preAuthCode,
					WxThirdPartyAuthConstants.PRE_AUTH_CODE_EXPIRED);
		} catch (ConnectException | SocketTimeoutException e) {
			LOGGER.error(e.toString());
		}
		return preAuthCode;
	}
```
注意：
1. @RequestMapping(value = "/wechat/redirectAuthPage")这个请求url来自第三方平台自身
2. resp.sendRedirect(WxThirdPartyAuthUtil.getRedirectAuthUrl(request));重定向的url中的redirect_uri参数对应第三方平台中的“**消息与事件接收URL**”参数