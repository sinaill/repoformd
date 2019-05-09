title: shiro验证与授权搭建
categories: shiro

---

### 导入包

```
		<!-- shiro核心包 -->
		<dependency>  
		    <groupId>org.apache.shiro</groupId>  
		    <artifactId>shiro-core</artifactId>  
		    <version>1.3.2</version>  
		</dependency>  
		<!-- 添加shiro web支持 -->
		<dependency>  
		    <groupId>org.apache.shiro</groupId>  
		    <artifactId>shiro-web</artifactId>  
		    <version>1.3.2</version>  
		</dependency>  
		<!-- 添加shiro spring整合 -->
		<dependency>  
		    <groupId>org.apache.shiro</groupId>  
		    <artifactId>shiro-spring</artifactId>  
		    <version>1.3.2</version>  
		</dependency>
		<dependency>
			<groupId>net.sf.ehcache</groupId>
			<artifactId>ehcache-core</artifactId>
			<version>2.4.3</version>
		</dependency>
		<dependency>
		    <groupId>org.apache.shiro</groupId>
		    <artifactId>shiro-ehcache</artifactId>
		    <version>1.3.2</version>
		</dependency>
```

### 配置文件

在spring配置文件中需要配置

```
	<!--shiro 核心-->
    <bean id="securityManager" class="org.apache.shiro.web.mgt.DefaultWebSecurityManager">
        <property name="cacheManager" ref="cacheManager"/>
        <!-- Single realm app.  If you have multiple realms, use the 'realms' property instead. -->
        <property name="sessionMode" value="native"/>
        <property name="realm" ref="ShiroRealm"></property>
		<!--或者可以配置多个realm-->
        <property name="realms">
        	<list>
    			<ref bean="jdbcRealm"/>
    			<ref bean="secondRealm"/>
    		</list>
        </property>
		
		<!--可以设置多realm验证策略,realm可以在下面authenticator中注入，但使用到授权时-->
		<!--只能在securityManager中注入-->
		<property name="authenticator" ref="authenticator"></property>
    </bean>
	
    <bean id="authenticator" 
    	class="org.apache.shiro.authc.pam.ModularRealmAuthenticator">
    	<property name="authenticationStrategy">
    		<bean class="org.apache.shiro.authc.pam.AtLeastOneSuccessfulStrategy"></bean>
    	</property>
    </bean>
	
	<!--缓存管理-->
    <bean id="cacheManager" class="org.apache.shiro.cache.ehcache.EhCacheManager">
        <!-- Set a net.sf.ehcache.CacheManager instance here if you already have one.  If not, a new one
             will be creaed with a default config:
             <property name="cacheManager" ref="ehCacheManager"/> -->
        <!-- If you don't have a pre-built net.sf.ehcache.CacheManager instance to inject, but you want
             a specific Ehcache configuration to be used, specify that here.  If you don't, a default
             will be used.: -->
        <property name="cacheManagerConfigFile" value="classpath:ehcache.xml"/>
    </bean>
	<!--注入自定义realm-->
    <bean id="ShiroRealm" class="crj.mspro.realm.ShiroRealm">
		<!--匹配器-->
		<property name="credentialsMatcher">
			<bean class="org.apache.shiro.authc.credential.HashedCredentialsMatcher">
				<property name="hashAlgorithmName" value="MD5"></property>
				<property name="hashIterations" value="1024"></property>
			</bean>
		</property>
    </bean>

	<!-- Shiro生命周期处理器-->
	<bean id="lifecycleBeanPostProcessor" class="org.apache.shiro.spring.LifecycleBeanPostProcessor"/>
	
	<!--自定义LogoutFilter,退出-->
    <bean id="logoutFilter" class="org.apache.shiro.web.filter.authc.LogoutFilter">
        <property name="redirectUrl" value="/main.jsp"/>
    </bean>

	<!--配置 ShiroFilter-->
	<!--id 必须和 web.xml 文件中配置的 DelegatingFilterProxy 的 <filter-name> 一致.
  	若不一致, 则会抛出: NoSuchBeanDefinitionException. 因为 Shiro 会来 IOC 容器中查找和 <filter-name> 名字对应的 filter bean.-->
    <bean id="shiroFilter" class="org.apache.shiro.spring.web.ShiroFilterFactoryBean">
        <property name="securityManager" ref="securityManager"/>
		<!--进入需要验证页面时跳转-->
        <property name="loginUrl" value="/main.jsp"/>
        <property name="successUrl" value="/main.jsp"/>
        <property name="unauthorizedUrl" value="/unauthorized.jsp"/>
        <!-- The 'filters' property is not necessary since any declared javax.servlet.Filter bean
             defined will be automatically acquired and available via its beanName in chain
             definitions, but you can perform overrides or parent/child consolidated configuration
             here if you like: -->
         <property name="filters">
            <map>
				<!--修改默认过滤器-->
                <entry key="logout" value-ref="logoutFilter"/>
            </map>
        </property> 

        <!--  
        	配置哪些页面需要受保护. 
        	以及访问这些页面需要的权限. 
        	1). anon 可以被匿名访问
        	2). authc 必须认证(即登录)后才可能访问的页面. 
        	3). logout 登出.
        	4). roles 角色过滤器
        -->
        <property name="filterChainDefinitions">
            <value>
            	/logout = logout
            	/easyui-jquery/** = anon
            	/main.jsp = anon
            	/js/** = anon
                /login = anon
                /css/** = anon
                /** = authc
            </value>
        </property>
		<!--此变量可以用实例工厂方法注入-->
		
		
    </bean>
```

工厂类

```
public class FilterChainDefinitionMapBuilder {

	public LinkedHashMap<String, String> buildFilterChainDefinitionMap(){
		LinkedHashMap<String, String> map = new LinkedHashMap<>();
		
		map.put("/login.jsp", "anon");
		map.put("/shiro/login", "anon");
		map.put("/shiro/logout", "logout");
		map.put("/user.jsp", "authc,roles[user]");
		map.put("/admin.jsp", "authc,roles[admin]");
		map.put("/list.jsp", "user");
		
		map.put("/**", "authc");
		
		return map;
	}
	
}

```

