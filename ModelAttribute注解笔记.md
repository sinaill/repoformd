title: ModelAttribute注解笔记
categories: 框架
tags: 
	- SpringMVC

---

### 介绍

@ModelAttribute.这个注解可以用在方法参数上，或是方法声明上。这个注解的主要作用是绑定request或是form参数到模型对象。可以使用保存在request或session中的对象来组装模型对象。注意，被@ModelAttribute注解的方法会在controller方法（@RequestMapping注解的）之前执行。因为模型对象要先于controller方法之前创建。

### 笔记

```
	@ModelAttribute
	public Person mda() {
		return new Person("zh", 14);
	}

	@RequestMapping("/test")
	public ModelAndView test(@ModelAttribute Person person,Model model) {
		ModelAndView mov = new ModelAndView("param");
		System.out.println(model.containsAttribute("person"));
		mov.addObject("person", person);
		return mov;
	}

	//或者

	@ModelAttribute("attr")
	public void mda(Map map) {
//		Person person = new Person("li", 12);
		map.put("person", new Person("zh", 14));
	}
```

1. 注解可以指定名称，如果没有指定，则以类名首字母小写为名字，若注解指定了名字，则在形参前面的ModelAttribute也要带上已指定的名字
2. 在ModelAttribute中使用Map,Model保存的数据，可以在controller中的Map或Model获取
3. 主要用来填充表格，当jsp页面表单项不包含该类所有属性时，可以先在ModelAttribute中根据相应变量先从数据库取出数据，然后表单数据提交到controller中对应方法时，覆盖用ModelAttribute注解修饰的形参，达到自动填充形参所有数据
4. [关于SessionAttribute,当Model中找不到时转到Session中查找](https://blog.csdn.net/m0_37893932/article/details/78327972)