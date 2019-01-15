# 公众号授权给第三方平台的技术实现流程概览
![：时序图](https://res.wx.qq.com/op_res/g360EANvw_kVk3WCt-rRVP5UNFVJ2pYjH6gQCmxVL58lWhow97U8wYXpB4gw-I-d)
# 推送component_verify_ticket协议
在第三方平台创建审核通过后，微信服务器会向其“授权事件接收URL”每隔10分钟定时推送component_verify_ticket。第三方平台方在收到ticket推送后也需进行解密），接收到后必须直接返回字符串success。
web层代码如下：
```
/**
	 * 授权事件接收
	 * 
	 * @param request
	 * @param response
	 */
	@RequestMapping(value = "/wechat/auth")
	@ResponseBody
	public void acceptAuthorizeEvent(HttpServletRequest request, HttpServletResponse response)
			throws IOException, AesException {
		response.getWriter().print("success");
		WxThirdPartyAuthUtil.processAuthorizeEvent(request);

	}

```
WxThirdPartyAuthUtil类的相关代码：

```
/**
	 * 处理授权事件的推送
	 * 
	 * @param request
	 * @throws IOException
	 * @throws AesException
	 * @throws DocumentException
	 */
	public static void processAuthorizeEvent(HttpServletRequest request) throws IOException, AesException {
		String nonce = request.getParameter("nonce");
		String timestamp = request.getParameter("timestamp");
		String msgSignature = request.getParameter("msg_signature");
		StringBuilder sb = new StringBuilder();
		BufferedReader in = request.getReader();
		String line;
		while ((line = in.readLine()) != null) {
			sb.append(line);
		}
		String xml = sb.toString();
		if(StringUtils.isEmpty(xml)){
			LOGGER.error("第三方平台全网发布-----------------------原始 Xml为空！");
			throw new CommonException("第三方平台全网发布-----------------------原始 Xml为空！");
		}
		LOGGER.error("第三方平台全网发布-----------------------原始 Xml=" + xml);
		WXBizMsgCrypt pc = new WXBizMsgCrypt(component_token, component_encodingaeskey, component_appid);
		String xml1 = pc.decryptMsg(msgSignature, timestamp, nonce, xml);
		LOGGER.error("第三方平台全网发布-----------------------解密后 Xml=" + xml1);
		processAuthorizationEvent(xml1);
	}

	/**
	 * 保存Ticket
	 * 
	 * @param xml
	 */
	static void processAuthorizationEvent(String xml) {
		Document doc;
		try {
			doc = DocumentHelper.parseText(xml);
			Element rootElt = doc.getRootElement();
			String ticket = rootElt.elementText("ComponentVerifyTicket");
			redisCache.put(WxThirdPartyAuthConstants.COMPONENT_VERIFY_TICKET
					, ticket);
			LOGGER.info("第三方平台全网发布-----------------------解密后 ComponentVerifyTicket=" + ticket);
		} catch (DocumentException e) {
			LOGGER.error(e.toString());
		}
	}
```
注意问题：
1. AES解密报错:Illegal key size,把jdk升级到jdk8u191（也可以去oracle官网下载jdk版本对应的jar包）
2. component_verify_ticket是官方推送过来的，可存到redis（建议不要设置过期时间），根据官方文档所述，其有效期比component_access_token（2小时）长，也并未准确定义它的有效时间。
3. RequestMapping(value = "/wechat/auth")这里的请求url对应第三方平台设置中的“**授权事件接收URL**”