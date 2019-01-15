第三方平台component_access_token是第三方平台的下文中接口的调用凭据，也叫做令牌（component_access_token）。每个令牌是存在有效期（2小时）的，且令牌的调用不是无限制的，请第三方平台做好令牌的管理，在令牌快过期时（比如1小时50分）再进行刷新。

代码如下：

```
/**
	 * 获取第三方平台访问公众号token
	 * @return
	 */
	public static String getComponentAccessToken() {
		String componentAccessToken = (String) redisCache.get(WxThirdPartyAuthConstants.COMPONENT_ACCESS_TOKEN);
		/** 判断数据库中是否存在component_access_token */
		if (StringUtils.isNotEmpty(componentAccessToken)) {
			/** 如果存在，直接返回token的值 */
			return componentAccessToken;
		}
		/** ticket若不存在，返回错误信息 */
		String componentVerifyTicket = (String) redisCache.get(WxThirdPartyAuthConstants.COMPONENT_VERIFY_TICKET);
		if (StringUtils.isEmpty(componentVerifyTicket)) {
			/** 错误信息 */
			throw new CommonException("如果出现这条消息，说明ticket以及token的过期处理存在BUG");
		}
		/** 拼装待发送的Json */
		JSONObject json = new JSONObject();
		json.put("component_appid", component_appid);
		json.put("component_appsecret", component_secret);
		json.put("component_verify_ticket", componentVerifyTicket);
		LOGGER.info("ThirdPartyServiceImpl:getComponentAccessToken:json={}", json);
		/** 发送Https请求到微信 */
		JSONObject resultJson;
		try {
			resultJson = HttpClientUtil.postJson(
					"https://api.weixin.qq.com/cgi-bin/component/api_component_token", json);
			LOGGER.error("ComponentAccessToken返回的数据"+resultJson.toString());
			/** 在返回结果中获取token */
			componentAccessToken = resultJson.getString(WxThirdPartyAuthConstants.COMPONENT_ACCESS_TOKEN);
			/** 保存token，并设置有效时间 */
			redisCache.put(WxThirdPartyAuthConstants.COMPONENT_ACCESS_TOKEN, 
					componentAccessToken,WxThirdPartyAuthConstants.COMPONENT_ACCESS_TOKEN_EXPIRED);
		} catch (Exception e) {
			LOGGER.error(e.toString());
		}

		return componentAccessToken;
	}
```
这部分较为简单，这里就不详细说明了。