---
title: Java泛型
date: 2017-03-01 17:05:01
tags: [Java, 泛型]
categories:
- Java
---

### 泛型是什么
最明显的例子是容器类，容器类需要为各种各样的类服务。倘若使用Object类来构建容器类，那么很容易出现类型转换的问题。泛型提供了强制类型的方法。

### 泛型怎么用
类型参数用作占位符，在运行时为类分配类型。根据需要，可能有一个或多个类型参数，并且可以用于整个类。根据惯例，类型参数是单个大写字母，该字母用于指示所定义的参数类型。下面列出每个用例的标准类型参数：

* E：元素
* K：键
* N：数字
* T：类型
* V：值
* S、U、V 等：多参数情况中的第 2、3、4 个类型

### 有界类型
我们经常会遇到这种情况，需要指定泛型类型，但希望控制可以指定的类型，而非不加限制。有界类型 在类型参数部分指定 extends 或 super 关键字，分别用上限或下限限制类型，从而限制泛型类型的边界。例如：

	<T extends UpperBoundType>
	
### 泛型方法
有时，我们可能不知道传入方法的参数类型。在方法级别应用泛型可以解决此类问题。方法参数可以包含泛型类型，方法也可以包含泛型返回类型。
```Java
	public static <N extends Number> double add(N a, N b){
	    double sum = 0;
	    sum = a.doubleValue() + b.doubleValue();
	    return sum;
	}
```

### 通配符
某些情况下，编写指定未知类型的代码很有用。问号 (?) 通配符可用于使用泛型代码表示未知类型。通配符可用于参数、字段、局部变量和返回类型。但最好不要在返回类型中使用通配符，因为确切知道方法返回的类型更安全。
```Java
	public static <T> void checkList(List<?> myList, T obj){
	    if(myList.contains(obj)){
	        System.out.println("The list contains the element: " + obj);
	    } else {
	        System.out.println("The list does not contain the element: " + obj);
	    }
	}
	
	public static <T> void checkNumber(List<? extends Number> myList, T obj){
	    if(myList.contains(obj)){
	        System.out.println("The list " + myList + " contains the element: " + obj);
	    } else {
	        System.out.println("The list " + myList + " does not contain the element: " + obj);
	    }
	}
```
