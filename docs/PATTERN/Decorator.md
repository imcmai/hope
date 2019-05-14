#### 0x01 前言
***
在编码的时候，我们为了扩展一个类常用的方法就是继承，随着扩展功能越来越多，子类会变得越来越膨胀，
使系统变得不灵活。

装饰器模式（Decorator pattern）允许像一个现有的对象添加新功能，同时又不改变其结构，他能让我们在扩展类的同时，系统保持较好的灵活性。

 ####0x02 废话不多说，进入正题，从一个场景开始
***
假设我们在北京有一块地，在这块地上我们要盖有好几间房间的别墅，每个房间的装修费用都不同，现在呢，我们需要对每个房间的费用进行计算并计算出别墅的价格。

```
//定义一个land表示这块地
public interface Land {
	 // 定义盖别墅需要花钱的规则
	Integer cost();
}

```
那么land定义好了在这块地上花钱的规则，但是盖一间房需要多少钱呢？

此时我们定义一个Room类，这个类定义了一间房间需要建造的基本费用。
```
public class Room implements Land{
	
	protected final int money = 1000 ;

	@Override
	public Integer cost() {
		return money ;
	}

}
```
开始建造房间，房间建造了两间，一间客厅(LivingRoom)，一间餐厅(DiningRoom)，建造完之后发现偌大房间连件家具都没有，这时候，我们计算一间房成本的时候需要加上家具的价格，那么，上代码...
```
public class LivingRoom extends Room{
    // 客厅
	@Override
	public Integer cost() {
		return this.land.cost()+200;
	}
}

public class DiningRoom extends Room{
    // 餐厅
	@Override
	public void cost() {
		return this.land.cost()+100;
	}
}
```
所以机智如你很容易就得到了建造一间客厅和一间餐厅的花费..
```
public static void main(String[] args) {
		LivingRoom livingRoom = new LivingRoom();
		System.out.println("建造一间客厅需要花费"+livingRoom.cost());
		DiningRoom diningRoom = new DiningRoom();
		System.out.println("建造一间餐厅需要花费"+diningRoom.cost());
	}
// 得到结果
建造一间客厅需要花费1200
建造一间餐厅需要花费1100
```
那么，问题来了：挖掘机...（啪，好好说啊，挖掘机是什么鬼啊，喂）emmmmm，好接着说：
####0x03 问题的产生
***
这么做实现确实很容易，虽然我们很容易就得到了建造一间客厅和一间餐厅需要的花费，但是这样的结构不具备灵活性，如果....我是说如果我的地比较小不能同时建造两间房子，只能把客厅和餐厅建在同一个屋子里，那么该去怎么计算费用？难道，还要很麻烦得去创建一个包含LivingDiningRoom类吗？这样做除了麻烦，而且还会使代码重复。

既然提出了问题，那么，肯定有解决的办法，怎么解决呢？
欲知后事如何..(pia，知你妹啊，还来？)，好吧，下面是解决办法。
####0x04 解决问题
***
（敲黑板！！）重点来了，为了更好的解决问题，我们需要做一些调整，同样先声明Land接口（抽象构件，是具体构件和抽象构件的共同父类）和Room类( ConcreteComponent ，具体构件，抽象构件的子类，装饰器可以额外添加职责。)，不同的是引入房间的一个装饰类RoomDecorator他实现了Land接口，因为没有实现Land的cost方法，所以需要声明为抽象类，并且定义一个Land接口为参数的构造函数，传入对象会保存在land中，该属性修饰符为protected方便子类访问。（划重点，考试要考）
```
public abstract class RoomDecorator implements Land{
	// 修饰符protected子类访问
	protected Land land ;
	
	public RoomDecorator(Land land) {  // 参数为实现Land接口的所有子类
		this.land = land ;
	}
}
```
我们重新定义客厅类和餐厅类
```
public class LivingRoom extends RoomDecorator{

	public LivingRoom(Land land) {
		super(land);
	}

	@Override
	public Integer cost() {
		return this.land.cost()+200;
	}
}

public class DiningRoom extends RoomDecorator{

	public DiningRoom(Land land) {
		super(land);
	}

	@Override
	public Integer cost() {
		return this.land.cost()+100;
	}
}
```
这两个类都继承了RoomDecorator，这意味着它们同时拥有指向Land接口的引用，当它们的cost()方法被调用时，都会先调用实现了Land接口的子类的cost()方法，然后执行自己特有的操作。

所以这时候建造一间客厅和一间餐厅所需的费用是这么计算的：
```
public static void main(String[] args) {
		LivingRoom livingRoom = new LivingRoom(new Room());
		System.out.println("建造一间客厅需要花费"+livingRoom.cost());
		
		DiningRoom diningRoom = new DiningRoom(new Room());
		System.out.println("建造一间餐厅需要花费"+diningRoom.cost());
	}
// 输出
建造一间客厅需要花费1200
建造一间餐厅需要花费1100
```
接着回到刚才的问题：建造一间有客厅又有餐厅的房间所需费用，代码如下
```
public static void main(String[] args) {
		LivingRoom diningLivingRoom = new LivingRoom(new DiningRoom(new Room()));
		System.out.println("建造一间客厅和餐厅需要花费"+diningLivingRoom.cost());
	}
// 输出
建造一间客厅和餐厅需要花费1300
```
我们计算建造费用的思路是：我们计算一间房的基础费用-->在基础房间装饰为客厅的费用-->在客厅的基础上加上装饰餐厅的费用-->得到包含客厅餐厅房间的费用，已经不需要通过麻烦的创建LivingDiningRoom类来计算包含餐厅客厅房间的建造费用了。

这便是装饰模式，通过一层层的装饰得到我们可以灵活的得到想要的结果，可以轻松地添加组件或者装饰类来创建灵活的结构。

####0x05 完整代码
```
//定义一个land表示这块地
//  Component：组件对象的接口，可以给这些对象动态的添加职责
public interface Land {
	 // 定义盖别墅需要花钱的规则
	Integer cost();
}
// ConcreteComponent：具体的组件对象，实现了组件接口。该对象通常就是被装饰器装饰的原始对象，可以给这个对象添加职责；
public class Room implements Land{
	protected final int money = 1000 ;
	@Override
	public Integer cost() {
		return money;
	}
}

// Decorator：所有装饰器的父类，需要定义一个与组件接口一致的接口
public abstract class RoomDecorator implements Land{
	// 修饰符protected子类访问
	protected Land land ;
	public RoomDecorator(Land land) {
		this.land = land ;
	}
}

// 装饰类具体构件
public class LivingRoom extends RoomDecorator{
	public LivingRoom(Land land) {
		super(land);
	}
	@Override
	public Integer cost() {
		return this.land.cost()+200;
	}
}
// 装饰类具体构件
public class DiningRoom extends RoomDecorator{
	public DiningRoom(Land land) {
		super(land);
	}
	@Override
	public Integer cost() {
		return this.land.cost()+100;
	}
}

// 测试
public class RoomTest{
    public static void main(String[] args) {
		LivingRoom livingRoom = new LivingRoom(new Room());
		System.out.println("建造一间客厅需要花费"+livingRoom.cost());
		
		DiningRoom diningRoom = new DiningRoom(new Room());
		System.out.println("建造一间餐厅需要花费"+diningRoom.cost());
		
		LivingRoom diningLivingRoom = new LivingRoom(new DiningRoom(new Room()));
		System.out.println("建造一间客厅和餐厅需要花费"+diningLivingRoom.cost());
	}
}
```
#### 0x06 The End...
***
>Tips:在北京有块地一辈子都不可能的，如果可以，我愿意偷电动车养你，点颗心在走嘛。