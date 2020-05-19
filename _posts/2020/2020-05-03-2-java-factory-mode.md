---
layout: post
title: 23种设计模式之工厂模式
category: java-design
tags: [java-design]
keywords: java
excerpt: 简单工厂模式，工厂方法模式，抽象工厂模式
lock: noneed
---

<mark>工厂模式</mark>.

简单工厂模式：一个抽象的接口，多个抽象接口的实现类，一个工厂类，用来实例化抽象的接口，代码如下：

```java
package com.jude.factory;

/**
 * @Author jude
 * @Date 2020/5/4 5:35 PM
 * @Version 1.0
 */

public interface Car {
	 void run();

	 void stop();
}

class Benz implements Car{

	@Override
	public void run() {
		System.out.println("Benz 开始启动了...");
	}

	@Override
	public void stop() {
		System.out.println("Benz 停车了...");
	}
}

class Ford implements Car {
	@Override
	public void run() {

	}

	@Override
	public void stop() {

	}
}

// 工厂类
class Factory {
	public static Car getCarInstance(String type){
		Car c = null;
		if("Benz".equals(type)){
			c =  new Benz();
		}else if("Ford".equals(type)){
			c = new Ford();
		}
		return c;
	}
}

class Test {
	public static void main(String[] args) {
		Car c = Factory.getCarInstance(("Benz"));
		if(c != null){
			c.run();
			c.stop();
		}else{
			System.out.println("造不类这种汽车");
		}
	}
}
```

工厂方法模式：有四个角色，抽象工厂模式，具体工厂模式，抽象产品模式，具体产品模式。不再是由一个工厂类去实例化具体的产品，而是由抽象工厂的子类去实例化产品，代码如下：

```java
// 抽象产品
public interface Moveable {
	void run();
}

// 具体产品
class Plane implements Moveable {
	@Override
	public void run() {
		System.out.println("plane...");
	}
}

// 具体产品
class Broom implements Moveable {
	@Override
	public void run() {
		System.out.println("broom...");
	}
}

// 抽象工厂
abstract class VehicleFactory {
	abstract Moveable create();
}

// 具体工厂
class PlaneFactory extends VehicleFactory {
	@Override
	Moveable create() {
		return new Plane();
	}
}

class BroomFactory extends VehicleFactory {
	@Override
	Moveable create() {
		return new Broom();
	}
}

// 测试
class TestFactory2 {
	public static void main(String[] args) {
		VehicleFactory factory = new BroomFactory();
		Moveable m = factory.create();
		m.run();
	}
}
```

抽象工厂模式：与工厂方法模式不同的是，工厂方法模式中的工厂只生产单一的产品，而抽象工厂模式中的工厂生产多个产品

```java
public  abstract class AbstractFactory {
  public abstract Vehicle createVhick();
  public abstract Weapon createWeapon();
  public abstract Food createFood();
}

// 具体工厂类，Vehicle,Weapon，Foood是抽象类
class DefaultFactory extends Abstractory {
  @Override
  public Food createFood() {
    return new Apple();
  }
  
  @Override
  public Food createWeapon() {
    return new AK47();
  }
  
  @Override
  public Food createVhick() {
    return new Car();
  }
}

// 测试类
class Test {
  public static void main(String[] args) {
    Abstractory f = new DefaultFactory();
    Vehicle v = f.createVhick();
    v.run();
    Weapon w = f.createWeapon();
    Food food = f.createFood();
  }
}
```

