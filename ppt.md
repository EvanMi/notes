# ppt





```java
public class GetAppStartInfoResponse {
    /**
     * 用来标识是否存在有效的授权信息，
     * 如果存在为true，不存在为false
     */
    private Boolean authorized;
    /**
     * 以下信息在authorized为true时有效
     * 唯一的code码，用来获取access_token
     */
    private String code;
    /**回调地址，该值由示例3.14传递*/
    private String redirectUrl;
    /**state，该值由示例3.14传递*/
    private String state;
    /**
     * 以下信息在authorized为false时有效。
     * 用户授权条目，用来显示用户在授权时要授权的权限信息
     */
    private List<String> terms;
    /**第三方应用在控制台系统中注册的应用图标*/
    private String clientImg;
    /**第三方应用在控制台系统中注册的应用名称*/
    private String clientName;
    /**Getters 和 Setters*/
}
```

