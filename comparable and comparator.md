title: comparable和comparator
categories: java基础
---

### comparable

集合中元素实现comparable，可以调用sort方法排序

```
class user implements Comparable<user>{
	private int id;
	private String name;
	
	public user() {
		super();
	}
	public user(int id, String name) {
		this.id = id;
		this.name = name;
	}
	/**
	 * @return the id
	 */
	
	
	public int getId() {
		return id;
	}
	/* (non-Javadoc)
	 * @see java.lang.Comparable#compareTo(java.lang.Object)
	 */
	@Override
	public int compareTo(user o) {
		// TODO Auto-generated method stub
		return this.id-o.id;
	}
	/**
	 * @param id the id to set
	 */
	public void setId(int id) {
		this.id = id;
	}
	/**
	 * @return the name
	 */
	public String getName() {
		return name;
	}
	/**
	 * @param name the name to set
	 */
	public void setName(String name) {
		this.name = name;
	}
	
}
```

调用Collections.sort(List<T> list)

```
		List<user> users = new ArrayList<user>();
		users.add(new user(1, "sdg"));
		users.add(new user(5, "dsf"));
		users.add(new user(2, "xzcv"));
		users.add(new user(4, "derg"));
		users.add(new user(3, "dfg"));
		Collections.sort(users);
		users.forEach(u->System.out.println(u.getId()));
```

输出
`1,2,3,4,5`

### comparator

当集合中元素没有实现comparable接口时，可以使用Collections.sort(List<T> list, Comparator<? super T> c)进行排序

```
		List<user> users = new ArrayList<user>();
		users.add(new user(1, "sdg"));
		users.add(new user(5, "dsf"));
		users.add(new user(2, "xzcv"));
		users.add(new user(4, "derg"));
		users.add(new user(3, "dfg"));
		Collections.sort(users,Comparator.comparing(user::getId));
		users.forEach(u->System.out.println(u.getId()));
```
输出
`1,2,3,4,5`