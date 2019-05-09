title: shiro授权
categories: shiro

---

### role和permission

授权类型分为role和permission，角色和权限permission粒度更小，通常可以为user:XXXX

### AuthorizingRealm

需要用到授权时，我们的realm可以继承自这个类

```
public class ShiroRealm extends AuthorizingRealm{


	protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principals) {

	}

	@Override
	protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token) throws AuthenticationException {

	}
	
}
```

`doGetAuthenticationInfo`用来验证

`doGetAuthorizationInfo`用来授权

查看授权方法返回值AuthorizationInfo的实现类，其中

```
    public void setRoles(Set<String> roles) {
        this.roles = roles;
    }

    public void addRole(String role) {
        if (this.roles == null) {
            this.roles = new HashSet<String>();
        }
        this.roles.add(role);
    }

    public void addRoles(Collection<String> roles) {
        if (this.roles == null) {
            this.roles = new HashSet<String>();
        }
        this.roles.addAll(roles);
    }

    public Set<String> getStringPermissions() {
        return stringPermissions;
    }

    public void addStringPermission(String permission) {
        if (this.stringPermissions == null) {
            this.stringPermissions = new HashSet<String>();
        }
        this.stringPermissions.add(permission);
    }

    public void addStringPermissions(Collection<String> permissions) {
        if (this.stringPermissions == null) {
            this.stringPermissions = new HashSet<String>();
        }
        this.stringPermissions.addAll(permissions);
    }
	·
	·
	·
```

所以我们可以这样实现进行授权

```
public class ShiroRealm extends AuthorizingRealm{


	protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principals) {

	}

	@Override
	protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token) throws AuthenticationException {
		
		SimpleAuthorizationInfo info = new SimpleAuthorizationInfo();
		
		info.addRole("admin");

		info.addStringPermission("user:update")
		
	}
	return info;
}
```