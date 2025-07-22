2023-01-15 03:44
Tags: #SpringSecurity #Oauth2
## OAuth2AuthorizationRequestRedirectFilter

**OAuth2AuthorizationRequestRedirectFilter** - фильтр отвечающий за первое перенаправление

По сути он берёт на себя 2 задачи:
  
1. Определить OAuth2AuthorizationRequest - сущность представляющая собой зарос к auth серверу на аутентификацию по протоколу oauth2
2.  Отправка этого запроса

``` JAVA
protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)  {  
     
	OAuth2AuthorizationRequest authRequest = this.authorizationRequestResolver
		.resolve(request);
    
    if (authorizationRequest != null) {  
         this.sendRedirectForAuthorization(request, response, authRequest);  
         return;      
    }  
}
```
### OAuth2AuthorizationRequest

Здесь есть всё необходимое для отправки запроса к серверу авторизации по протоколу oauth2

``` JAVA
public final class OAuth2AuthorizationRequest {
	private String authorizationUri;
	private AuthorizationGrantType authorizationGrantType;
	private OAuth2AuthorizationResponseType responseType;
	private String clientId;
	private String redirectUri;
	private Set<String> scopes;
	private String state;
	private Map<String, Object> additionalParameters;
	private String authorizationRequestUri;
	private Map<String, Object> attributes;
	..
}
```

Сама сущность запроса конечно не отправляет, а просто инкапсулирует в себе данные для запроса, *а также содержит удобный builder*

За получение такого объекта из запроса отвечает resolver, а именно **OAuth2AuthorizationRequestResolver**

### OAuth2AuthorizationRequestResolver

У единственной его реализации метод resolve выглядит следующим образом:

1.  Получает объект **ClientRegistration** исходя из переданного registrationId (представляет собой строку названия auth server-а, например google)

``` JAVA
private OAuth2AuthorizationRequest resolve(HttpServletRequest request, 
										   String registrationId,
										   String redirectUriAction) {

	ClientRegistration clientRegistration = this.clientRegistrationRepository.findByRegistrationId(registrationId);
	OAuth2AuthorizationRequest.Builder builder = getBuilder(clientRegistration);
	
	String redirectUriStr = expandRedirectUri(
		request, clientRegistration, redirectUriAction
	);
	  
	// @formatter:off
		builder.clientId(clientRegistration.getClientId())
			.authorizationUri(
				clientRegistration.getProviderDetails().getAuthorizationUri()
			)
			.redirectUri(redirectUriStr)
			.scopes(clientRegistration.getScopes())
			.state(DEFAULT_STATE_GENERATOR.generateKey());
	// @formatter:on
	
	this.authorizationRequestCustomizer.accept(builder);
	return builder.build();
}
```

## OAuth2LoginAuthenticationFilter

Используется при аутентификации пользователя, после redirect со стороны сервера авторизации.

Внутри этого фильтра и происходит запрос на получение **access токена** в spring, за получение этого токена отвечают 2 провайдера: 
- **OidcAuthorizationCodeAuthenticationProvider** 
- **OAuth2LoginAuthenticationProvider**

Исходя из их названия понятно, что каждый из них обрабатывает свой вариант протокола: **OIDC** и **Oauth2** соответственно.

А раз сигналом для auth сервера о протоколе OICD является содержание в **scope** значения **openId**, то именно по этому значению они и определят между собой, кто из них должен отработать.  


``` Java
// самое начало метода authenticate у OidcAuthorizationCodeAuthenticationProvider
if (!authorizationCodeAuthentication
			.getAuthorizationExchange()
			.getAuthorizationRequest()
			.getScopes()  
		    .contains(OidcScopes.OPENID)) {  
      return null;
}
```

*Access токен эти провайдеры достают используя обычный Spring RestClient, и отправляя соответcвующие запросы к auth серверу*

