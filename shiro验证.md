title: shiro验证
categories: shiro

---

### subject 

subject指的是:"当前正在执行的用户的特定的安全视图",可以把Subject看成是shiro的"User"概念

### login

验证从调用subject的login方法开始
```
	//controller中
	Subject currentUser = SecurityUtils.getSubject();
	if(!currentUser.isAuthenticated()) {
		UsernamePasswordToken token = new UsernamePasswordToken(userName, password);
		currentUser.login(token);

```

进入login方法

```
Subject subject = securityManager.login(this, token);
```

```
    public Subject login(Subject subject, AuthenticationToken token) throws AuthenticationException {
        AuthenticationInfo info;
        try {
            info = authenticate(token);
        } catch (AuthenticationException ae) {
            try {
                onFailedLogin(token, ae, subject);
            } catch (Exception e) {
                if (log.isInfoEnabled()) {
                    log.info("onFailedLogin method threw an " +
                            "exception.  Logging and propagating original AuthenticationException.", e);
                }
            }
            throw ae; //propagate
        }

        Subject loggedIn = createSubject(token, info, subject);

        onSuccessfulLogin(token, info, loggedIn);

        return loggedIn;
    }
```

查看`info = authenticate(token)`,可以看到这里抛出异常则验证失败

```
    public AuthenticationInfo authenticate(AuthenticationToken token) throws AuthenticationException {
        return this.authenticator.authenticate(token);
    }
```

在此处打断点debug可知调用以上方法的类实例为`org.apache.shiro.web.mgt.DefaultWebSecurityManager@5c2a5a3`，即在配置文件中已配置的

往下继续查看，类实例变为`org.apache.shiro.authc.pam.ModularRealmAuthenticator@10747cac`，也就是配置文件中的authenticator

```
info = doAuthenticate(token);
```

```
    protected AuthenticationInfo doAuthenticate(AuthenticationToken authenticationToken) throws AuthenticationException {
        assertRealmsConfigured();
		//获取authenticator中注入的realm
        Collection<Realm> realms = getRealms();
        if (realms.size() == 1) {
			//配置单realm时进入此方法
            return doSingleRealmAuthentication(realms.iterator().next(), authenticationToken);
        } else {
			//多realm
            return doMultiRealmAuthentication(realms, authenticationToken);
        }
```

还记得配置文件中，realm是可以注入在securitymanager中的，那这里是怎么获得authenticator中的realm呢，在authenticator中的`setRealms`方法打断点，debug之后发现，在初始化的时候shiro会将注入securitymanager中的realm通过set方法注入到authenticator中

进入`doSingleRealmAuthentication(realms.iterator().next(), authenticationToken);`

调用了我们自定义realm的`getAuthenticationInfo(token)`方法
```
AuthenticationInfo info = realm.getAuthenticationInfo(token);
```

进入`getAuthenticationInfo(token)`

在这个方法中，对密码进行匹配，失败时抛出异常，则登录验证失败

```
    public final AuthenticationInfo getAuthenticationInfo(AuthenticationToken token) throws AuthenticationException {

        AuthenticationInfo info = getCachedAuthenticationInfo(token);
        if (info == null) {
            //otherwise not cached, perform the lookup:
            info = doGetAuthenticationInfo(token);
            log.debug("Looked up AuthenticationInfo [{}] from doGetAuthenticationInfo", info);
            if (token != null && info != null) {
                cacheAuthenticationInfoIfPossible(token, info);
            }
        } else {
            log.debug("Using cached authentication info [{}] to perform credentials matching.", info);
        }

        if (info != null) {
            assertCredentialsMatch(token, info);
        } else {
            log.debug("No AuthenticationInfo found for submitted AuthenticationToken [{}].  Returning null.", token);
        }

        return info;
    }
```

注意
```
info = doGetAuthenticationInfo(token);
assertCredentialsMatch(token, info);
```
第一条调用了我们自定义realm的doGetAuthenticationInfo(token)，就是需要我们覆写的方法

第二条对客户端传入的token和数据库取出的账号数据作匹配(即`doGetAuthenticationInfo(token)`方法的返回值

先查看`assertCredentialsMatch(token, info);`方法

```
    protected void assertCredentialsMatch(AuthenticationToken token, AuthenticationInfo info) throws AuthenticationException {
        CredentialsMatcher cm = getCredentialsMatcher();
        if (cm != null) {
            if (!cm.doCredentialsMatch(token, info)) {
                //not successful - throw an exception to indicate this:
                String msg = "Submitted credentials for token [" + token + "] did not match the expected credentials.";
                throw new IncorrectCredentialsException(msg);
            }
        } else {
            throw new AuthenticationException("A CredentialsMatcher must be configured in order to verify " +
                    "credentials during authentication.  If you do not wish for credentials to be examined, you " +
                    "can configure an " + AllowAllCredentialsMatcher.class.getName() + " instance.");
        }
    }
```

`getCredentialsMatcher()`获取我们在配置文件中注入的`CredentialsMatcher`，此处配置的为`HashedCredentialsMatcher`，找到它的`doCredentialsMatch(token, info)`

```
    public boolean doCredentialsMatch(AuthenticationToken token, AuthenticationInfo info) {
        Object tokenHashedCredentials = hashProvidedCredentials(token, info);
        Object accountCredentials = getCredentials(info);
        return equals(tokenHashedCredentials, accountCredentials);
    }
```

其中`hashProvidedCredentials(token, info)`方法将我们传入的token，即客户端传来的账户信息，将其中密码加密

```
   protected Hash hashProvidedCredentials(Object credentials, Object salt, int hashIterations) {
        String hashAlgorithmName = assertHashAlgorithmName();
        return new SimpleHash(hashAlgorithmName, credentials, salt, hashIterations);
    }
```

然后跟数据库取出的账户密码比对，所以数据库中的密码需是同种加密过

最后我们在看`info = doGetAuthenticationInfo(token);`当我们只需要验证而不需要授权时，定义realm的时候直接继承AuthenticatingRealm然后实现`doGetAuthenticationInfo`方法返回数据库中的账号信息，realm中的`etAuthenticationInfo`方法会调用`doGetAuthenticationInfo`验证账号信息

```
public class MyRealm extends AuthenticatingRealm {

	@Override
	protected AuthenticationInfo doGetAuthenticationInfo(
			AuthenticationToken token) throws AuthenticationException {
		//第一个参数会存放在subject中，通过SecurityUtils.getSubject().getPrincipal()可取出
		//第二个参数为要匹配的密码，第三个盐值，第四个realm名
		info = new SimpleAuthenticationInfo(principle, credentials, credentialsSalt, realmName);
		return info;
	}
}
```

大致流程为subject.login->DefaultWebSecurityManager->ModularRealmAuthenticator->realm
主要验证过程是在realm的`getAuthenticationInfo`方法中











