title: ajax中json和springmvc
categories: 框架
tags: 
	- SpringMvc
	- ajax

---

### json

json对象格式:`{"param":"value","param":"value"}`

普通js对象:`{param:"value",param:"value"}`



js对象转json对象:

- 先将js转json字符串：`JSON.stringify()`
- 将json字符串转json对象:`JSON.parse()`

### ajax中的contentType

ajax中，没有指定`contentType`的话，Jquery默认使用`application/x-www-form-urlencoded`类型，data使用json对象格式,`springmvc`可以`@RequestParam`接收数据(变量同名不需要)

指定`contentType`为`application/json`时，data必须为json字符串格式，springmvc接收数据要用`@RequestBody`注解

