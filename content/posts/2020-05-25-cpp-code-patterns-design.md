---
title: "C++ 设计模式"
date: 2020-05-25T17:03:18+08:00
toc: true
description: C++ 设计模式 Wiki 翻译
tags: ["C++"]
---

注：一直不能把四人帮的设计模式看完，翻译一篇简短的 [wiki](https://en.wikibooks.org/wiki/C%2B%2B_Programming/Code/Design_Patterns) 帮助理解。

## 创建型模式

在软件工程中，**创建型模式** （Creational Patterns）是一种用于解决对象创建机制的设计模式，目的为使用对情景合适的创建对象的方法。最简单的创建对象的方法可能导致设计问题或使设计变得更复杂。创建型模式通过控制对象的创建来解决该问题。

在这一节，我们假设读者已经对函数、全局变量、栈与堆的区别、类、指针、静态成员函数的只是有所了解。

我们将了解的数种创建型模式分别解决特定的实现情景，会把代码抽象一个层次，以下逐个介绍。

### 构造者模式

**构造者模式**（Builder Pattern）是用来将一个复杂对象的构造和属性分开，这样能够通过同样的构造过程来创造出具有不同属性的对象。

#### 问题

在不使用一个复杂的构造函数或具有过多参数的函数的情况下，来构造一个复杂的对象。

#### 解决方案

定义一个在对象对用户可用前 使用成员函数对目标对象一步步构造 的中间层对象。构造者模式可以实现把构造动作推迟到所有的构造选项都已被指定时。

```c++
#include <iostream>
#include <memory>
class Pizza {
public:
  void setDough(const std::string &dough) { m_dough = dough; }
  void setSauce(const std::string &sauce) { m_sauce = sauce; }
  void setTopping(const std::string &topping) { m_topping = topping; }
  void open() const {
    std::cout << "The Pizza have " << m_dough << " dough, " << m_sauce
              << " sauce, " << m_topping << " topping." << std::endl;
  }

private:
  std::string m_dough;
  std::string m_sauce;
  std::string m_topping;
};

class PizzaBuilder {
public:
  virtual ~PizzaBuilder() = default;
  void createNewPizza() { m_pizza = std::make_unique<Pizza>(); }
  Pizza *getPizza() { return m_pizza.release(); }
  virtual void buildDough() = 0;
  virtual void buildSauce() = 0;
  virtual void buildTopping() = 0;

protected:
  std::unique_ptr<Pizza> m_pizza;
};

class HawaiianPizzaBuilder : public PizzaBuilder {
public:
  ~HawaiianPizzaBuilder() override = default;
  void buildDough() override { m_pizza->setDough("Hawaiian dough"); }
  void buildSauce() override { m_pizza->setSauce("Hawaiian sauce"); }
  void buildTopping() override { m_pizza->setTopping("Hawaiian topping"); }
};

class SpicyPizzaBuilder : public PizzaBuilder {
public:
  ~SpicyPizzaBuilder() override = default;
  void buildDough() override { m_pizza->setDough("Spicy dough"); }
  void buildSauce() override { m_pizza->setSauce("Spicy sauce"); }
  void buildTopping() override { m_pizza->setTopping("Spicy topping"); }
};

class Cook {
public:
  void openPizza() const { m_pizzaBuilder->getPizza()->open(); }
  void createPizza(PizzaBuilder *pizzaBuilder) {
    m_pizzaBuilder = pizzaBuilder;
    m_pizzaBuilder->createNewPizza();
    m_pizzaBuilder->buildDough();
    m_pizzaBuilder->buildSauce();
    m_pizzaBuilder->buildTopping();
  }

private:
  PizzaBuilder *m_pizzaBuilder;
};

int main() {
  Cook cook{};
  HawaiianPizzaBuilder hawaiianPizzaBuilder;
  cook.createPizza(&hawaiianPizzaBuilder);
  cook.openPizza();

  SpicyPizzaBuilder spicyPizzaBuilder;
  cook.createPizza(&spicyPizzaBuilder);
  cook.openPizza();
}
// 输出
// The Pizza have Hawaiian dough dough, Hawaiian sauce sauce, Hawaiian topping
// topping. The Pizza have Spicy dough dough, Spicy sauce sauce, Spicy topping
// topping.
```

### 工厂模式

可以创建同一基类的继承类的实例的工具类。

### 抽象工厂模式

可以创建数个不同基类的继承类的实例的工具类，这一工具类也可返回某一基类的工厂。

在需要创建同一基类多种不同继承类的对象的场景中，工厂模式很有用。工厂方法定义创建对象的方法，其继承类即可以重载来指定需要创建的不同的继承类。在运行时，可以将目标对象的描述符（如用户输入的字符串）传入工厂方法，调用并得到一个指向新被创建对象的基类指针。在基类的设计较好（不需要将基类指针转型到继承类）时，这种模式效果最好。

#### 问题

在运行时才能够根据配置或应用参数来决定创建的类的类型，写代码时不知道哪个继承类会被实例化。

#### 解决方案

定义一个创建对象的接口，使其继承类来决定实例化哪个类。工厂方法允许类在继承类中才被实例化。

如下定义抽象类 `Computer` 与其继承类：`Laptop` 和 `Desktop`。

```c++
class Computer {
public:
  virtual void Run() = 0;
  virtual void Stop() = 0;

  virtual ~Computer(){}; // 若无该行，该例中无法正确调用其继承类的构造函数
};
class Laptop : public Computer {
public:
  void Run() override { mHibernating = false; };
  void Stop() override { mHibernating = true; };
  virtual ~Laptop(){}; 
private:
  bool mHibernating; // 该设备正在 hibernate
};
class Desktop : public Computer {
public:
  void Run() override { mOn = true; };
  void Stop() override { mOn = false; };
  virtual ~Desktop(){};

private:
  bool mOn; // 该设备是否启动
};
```

给定描述， `ComputerFactory` 类返回某一继承类的实例。

```c++
class ComputerFactory {
public:
  static Computer *NewComputer(const std::string &description) {
    if (description == "laptop")
      return new Laptop;
    if (description == "desktop")
      return new Desktop;
    return nullptr;
  }
};
```

以下分析这种设计的优点。首先体现在编译上。这一模式允许将接口  `Computer`  声明在与工厂类独立的头文件中，我们便可以将  `NewComputer()`  的实现写在一个独立的独立的实现文件中。此时只有 `NewComputer()` 函数需要知道继承类的细节。这样，若 `Computer` 的继承类的代码发生改变，或者添加了新的继承类，只有 `NewComputer()` 所在的文件需要重新编译。使用工厂的用户只关注接口，在应用的生命周期内，接口应该保持一致性。

此外，如果需要添加一个继承类，用户通过用户接口来请求创建对象，之前调用工厂的代码不需要改变便可以添加新的 `Computer` 继承类。使用工厂的代码只需要将对应新继承类的字符串传至工厂，再让工厂允许处理该类即可。

假设需要编程一个视频游戏，将来可能添加新的怪的类型，每个怪都具有不同的 AI 函数并可以相异地更新画面。通过使用工厂方法，程序的控制器可以在不依赖于了解怪具体类型的情况下通过调用工厂来创建怪。此时，开发者将可以创建使用有新 AI 和新的更新画面函数的怪，将其添加至工厂，创建一个新的调用工厂的阶段，通过名字来创建怪。通过 XML 描述，开发者甚至可以不进行重新编译便可创建新的怪。而这一切，都归功于将类型创建和类型使用独立开来。

另一个例子：

```c++
#include <iostream>
#include <memory>
#include <stdexcept>
using namespace std;

class Pizza {
public:
  virtual int getPrice() const = 0;
  virtual ~Pizza(){};
};

class HamAndMushroomPizza : public Pizza {
public:
  virtual int getPrice() const { return 850; };
  virtual ~HamAndMushroomPizza(){};
};

class DeluxePizza : public Pizza {
public:
  virtual int getPrice() const { return 1050; };
  virtual ~DeluxePizza(){};
};

class HawaiianPizza : public Pizza {
public:
  virtual int getPrice() const { return 1150; };
  virtual ~HawaiianPizza(){};
};

class PizzaFactory {
public:
  enum PizzaType { HamMushroom, Deluxe, Hawaiian };

  static unique_ptr<Pizza> createPizza(PizzaType pizzaType) {
    switch (pizzaType) {
    case HamMushroom:
      return make_unique<HamAndMushroomPizza>();
    case Deluxe:
      return make_unique<DeluxePizza>();
    case Hawaiian:
      return make_unique<HawaiianPizza>();
    }
    throw "invalid pizza type.";
  }
};

/*
 * Create all available pizzas and print their prices
 */
void pizza_information(PizzaFactory::PizzaType pizzatype) {
  unique_ptr<Pizza> pizza = PizzaFactory::createPizza(pizzatype);
  cout << "Price of " << pizzatype << " is " << pizza->getPrice() << std::endl;
}

int main() {
  pizza_information(PizzaFactory::HamMushroom);
  pizza_information(PizzaFactory::Deluxe);
  pizza_information(PizzaFactory::Hawaiian);
}
```

### 原型模式

**原型模式**（Prototype Pattern）通常用于需要创建的类型由原型的实例来决定，通过复制来创建新的实例。这一模式通常用于构造继承类的代价比较高时。

#### 实现

创建抽象基类定义一个纯虚函数 `clone()`。需要多态构造函数的类继承自抽象基类，并实现 `clone()` 这一方法。

这样，客户的代码首先调用工厂方法。工厂方法根据参数来定位需要构造的类，然后调用该类中的 `clone()` 方法，将对象返回并传回客户。

以下是原型模式的简单的栗子。该例子中的组件如下：

- `Record` 类是一个纯虚类，其有一个纯虚方法 `clone()`。
- `CardRecord`，`BikeRecord`，`PersonRecord` 为 `Record` 类的实现类。
- 枚举类型 `RecordType` 的值映射为以上实现类。
- `RecordFactory` 有工厂方法 `CreateRecord(...)`，该方法的参数为 `RecordType` 值，根据这一参数来返回对应 `Record` 的实现。

```c++
#include <iostream>
#include <memory>
#include <string>
#include <unordered_map>
using namespace std;

// Record 是原型基类
class Record {
public:
  virtual ~Record() {}
  virtual void print() = 0;
  virtual unique_ptr<Record> clone() = 0;
};

// CarRecord 是一个实类原型
class CarRecord : public Record {
private:
  string m_carName;
  int m_ID;

public:
  CarRecord(string carName, int ID) : m_carName(carName), m_ID(ID) {}

  void print() override {
    cout << "Car Record" << endl
         << "Name  : " << m_carName << endl
         << "Number: " << m_ID << endl
         << endl;
  }

  unique_ptr<Record> clone() override { return make_unique<CarRecord>(*this); }
};

class BikeRecord : public Record {
private:
  string m_bikeName;
  int m_ID;

public:
  BikeRecord(string bikeName, int ID) : m_bikeName(bikeName), m_ID(ID) {}

  void print() override {
    cout << "Bike Record" << endl
         << "Name  : " << m_bikeName << endl
         << "Number: " << m_ID << endl
         << endl;
  }

  unique_ptr<Record> clone() override { return make_unique<BikeRecord>(*this); }
};

class PersonRecord : public Record {
private:
  string m_personName;
  int m_age;

public:
  PersonRecord(string personName, int age)
      : m_personName(personName), m_age(age) {}

  void print() override {
    cout << "Person Record" << endl
         << "Name : " << m_personName << endl
         << "Age  : " << m_age << endl
         << endl;
  }

  unique_ptr<Record> clone() override {
    return make_unique<PersonRecord>(*this);
  }
};

// 不透明的 Record 类型，避免直接暴露实现
enum RecordType { CAR, BIKE, PERSON };

// RecordFactory 为客户
class RecordFactory {
private:
  unordered_map<RecordType, unique_ptr<Record>, hash<int>> m_records;

public:
  RecordFactory() {
    m_records[CAR] = make_unique<CarRecord>("Ferrari", 5050);
    m_records[BIKE] = make_unique<BikeRecord>("Yamaha", 2525);
    m_records[PERSON] = make_unique<PersonRecord>("Tom", 25);
  }

  unique_ptr<Record> createRecord(RecordType recordType) {
    return m_records[recordType]->clone();
  }
};

int main() {
  RecordFactory recordFactory;

  auto record = recordFactory.createRecord(CAR);
  record->print();

  record = recordFactory.createRecord(BIKE);
  record->print();

  record = recordFactory.createRecord(PERSON);
  record->print();
}
```

另一个例子：

```c++
class CPrototypeMonster {
protected:
  CString _name;

public:
  CPrototypeMonster();
  CPrototypeMonster(const CPrototypeMonster &copy);
  virtual ~CPrototypeMonster();

  virtual CPrototypeMonster *
  Clone() const = 0; // This forces every derived class to provide an override
                     // for this function.
  void Name(CString name);
  CString Name() const;
};

class CGreenMonster : public CPrototypeMonster {
protected:
  int _numberOfArms;
  double _slimeAvailable;

public:
  CGreenMonster();
  CGreenMonster(const CGreenMonster &copy);
  ~CGreenMonster();

  virtual CPrototypeMonster *Clone() const;
  void NumberOfArms(int numberOfArms);
  void SlimeAvailable(double slimeAvailable);

  int NumberOfArms() const;
  double SlimeAvailable() const;
};

class CPurpleMonster : public CPrototypeMonster {
protected:
  int _intensityOfBadBreath;
  double _lengthOfWhiplikeAntenna;

public:
  CPurpleMonster();
  CPurpleMonster(const CPurpleMonster &copy);
  ~CPurpleMonster();

  virtual CPrototypeMonster *Clone() const;

  void IntensityOfBadBreath(int intensityOfBadBreath);
  void LengthOfWhiplikeAntenna(double lengthOfWhiplikeAntenna);

  int IntensityOfBadBreath() const;
  double LengthOfWhiplikeAntenna() const;
};

class CBellyMonster : public CPrototypeMonster {
protected:
  double _roomAvailableInBelly;

public:
  CBellyMonster();
  CBellyMonster(const CBellyMonster &copy);
  ~CBellyMonster();

  virtual CPrototypeMonster *Clone() const;

  void RoomAvailableInBelly(double roomAvailableInBelly);
  double RoomAvailableInBelly() const;
};

CPrototypeMonster *CGreenMonster::Clone() const {
  return new CGreenMonster(*this);
}

CPrototypeMonster *CPurpleMonster::Clone() const {
  return new CPurpleMonster(*this);
}

CPrototypeMonster *CBellyMonster::Clone() const {
  return new CBellyMonster(*this);
}

void DoSomeStuffWithAMonster(const CPrototypeMonster *originalMonster) {
  CPrototypeMonster *newMonster = originalMonster->Clone();
  ASSERT(newMonster);

  newMonster->Name("MyOwnMonster");
  // Add code doing all sorts of cool stuff with the monster.
  delete newMonster;
}
```

### 单例模式

**单例模式**（Singleton Pattern）保证类只有一个实例，并提供了指向该实例的全局变量指针。以只包含一个元素的单例集合命名（注：没理解）。这样的模式当一个对象需要在整个程序范围内共同使用时非常有用。

要件清单：

- 在单例类中定义一个私有静态成员。
- 在该类中定义一个公有静态访问函数。
- 该访问函数的实现使用”懒初始化“（在第一次被使用时创建）。
- 将所有的构造函数定义为受保护（protected）或私有。
- 客户只能够通过访问函数来操作单例。

单例与其他变量类型的区别：单例位于任何函数之外。传统的实现以单例类的静态成员函数的方式，第一次调用时创建一个单例类的实体，以后的调用都一直访问该实体。以下为代码示例。

```c++
#include <memory>
#include <string>

class StringSingleton {
public:
  // 类自身的一些访问函数
  std::string GetString() const { return mString; }
  void SetString(const std::string &newStr) { mString = newStr; }

  // 该静态函数允许从任何地方访问，欲获取该类的实例，可调用：
  //     StringSingleton::Instance().GetString();
  static StringSingleton &Instance() {
    // 这一行仅运行一次，保证仅创建单个对象
    static auto instance = std::make_unique<StringSingleton>();
    // 对指针取值，使调用者不必使用方向操作符，也可以避免调用者尝试删除变量（笑
    return *instance; // 永远返回同一实例
  }

private:
  // 需要将默认函数私有化避免被客户调用
  StringSingleton() {} // 默认的构造函数只对成员和友元类可见

  // 禁止调用以下两个函数
  StringSingleton(const StringSingleton &old) = delete;
  const StringSingleton &operator=(const StringSingleton &old) = delete;

  // 一些编译器可能不会默认将析构函数私有，这样可以防止客户删除
  ~StringSingleton() {}

private:
  std::string mString;
};
```

TODO：讨论 Meyers 单例与其他各种单例

单例的应用：

最通用之一为应用配置。配置信息可能需要被全局访问，并且可能将需要扩展。最方便的替代方式是使用全局 `struct`，但这样无法明确对象何时被创建，也不无法保证对是否存在。

例如，另一个开发者在其对象的构造函数中使用你的单例，然后又一个开发者决定在全局范围内创造这一在构造函数内使用单例的实体。如果只是简单地使用了全局变量，那么编译过程中链接的顺序很重要。在 main 执行前，无法确定全局变量初始化和使用全局变量类初始化的顺序。这会对其他代码的执行造成错误，却很难调试。然而，使用单例模式，在对象第一次被访问时，配置对象即会被创建，这一对象将在程序的生命周期内一直存在，之后被使用时它不会不存在。

另一个常见应用为更新旧代码使其在新的架构中也可以工作。由于开发者可以任意地使用全局变量，将其移入一个类中，并使其为单例，单例可以作为媒介层内联至梗强壮的面向对象的结构。

```c++
#include <iostream>
using namespace std;

// Mutex 实现
class Mutex {};

// 锁的实现
class Lock {
public:
  Lock(Mutex &m) : mutex(m) { // 获得锁的实现
  }
  ~Lock() { // 释放锁的实现
  }

private:
  Mutex &mutex;
};

class Singleton {
public:
  static Singleton *GetInstance();
  int a;
  ~Singleton() { cout << "In Destructor" << endl; }

private:
  Singleton(int _a) : a(_a) { cout << "In Constructor" << endl; }

  static Mutex mutex;

  Singleton(const Singleton &);
  Singleton &operator=(const Singleton &other);
};

Mutex Singleton::mutex;

Singleton *Singleton::GetInstance() {
  Lock lock(mutex);

  cout << "Get Instance" << endl;

  // 第一次被访问时初始化
  static Singleton inst(1);

  return &inst;
}

int main() {
  Singleton *singleton = Singleton::GetInstance();
  cout << "The value of the singleton: " << singleton->a << endl;
  return 0;
}
```

## 结构性模式

### 适配器模式

将一个类的接口转化为另一个用户期待的接口。**适配器**（Adpater）使得接口不兼容的类能够在一起工作。

```c++
#include <iostream>

class Dog { // 抽象目标
public:
  virtual ~Dog() = default;
  virtual void performsConversion() const = 0;
};

class DogFemale : public Dog { // 实现实体目标
public:
  virtual void performsConversion() const override {
    std::cout << "Dog female performs conversion." << std::endl;
  }
};

class Cat { // 抽象适配者
public:
  virtual ~Cat() = default;
  virtual void performsConversion() const = 0;
};

class CatFemale : public Cat {
public:
  virtual void performsConversion() const override {
    std::cout << "Cat female performs conversion." << std::endl;
  }
};

class DogNature {
public:
  void carryOutNature(Dog *dog) {
    std::cout << "On with the Dog nature!" << std::endl;
    dog->performsConversion();
  }
};

class ConversionAdapter : public Dog { // 适配者
private:
  Cat *cat;

public:
  ConversionAdapter(Cat *c) : cat(c) {}
  virtual void performsConversion() const override {
    cat->performsConversion();
  }
};

int main() { // 用户代码
  DogFemale *dogFemale = new DogFemale;
  CatFemale *catFemale = new CatFemale;
  DogNature dogNature;
  //	dogNature.carryOutNature (catFemale);
  //	当然，无法编译由于其接收的类型必须为 Dog*
  ConversionAdapter *adaptedCat =
      new ConversionAdapter(catFemale); // catFemale 被适配为 Dog

  dogNature.carryOutNature(dogFemale);
  dogNature.carryOutNature(adaptedCat);

  delete adaptedCat;
  delete catFemale;
  delete dogFemale;
  return 0;
}
```

### 桥接模式

**桥接模式**（Bridge Pattern）用于将接口与实现分离。这样可以保证灵活性，保证接口与实现可以独立地改变。

以下示例输出：
```
API1.circle at 1:2 7.5
API2.circle at 5:7 27.5
```

```c++
#include <iostream>

using namespace std;

/* Implementor*/
class DrawingAPI {
public:
  virtual void drawCircle(double x, double y, double radius) = 0;
  virtual ~DrawingAPI() {}
};

/* Concrete ImplementorA*/
class DrawingAPI1 : public DrawingAPI {
public:
  void drawCircle(double x, double y, double radius) {
    cout << "API1.circle at " << x << ':' << y << ' ' << radius << endl;
  }
};

/* Concrete ImplementorB*/
class DrawingAPI2 : public DrawingAPI {
public:
  void drawCircle(double x, double y, double radius) {
    cout << "API2.circle at " << x << ':' << y << ' ' << radius << endl;
  }
};

/* Abstraction*/
class Shape {
public:
  virtual ~Shape() {}
  virtual void draw() = 0;
  virtual void resizeByPercentage(double pct) = 0;
};

/* Refined Abstraction*/
class CircleShape : public Shape {
public:
  CircleShape(double x, double y, double radius, DrawingAPI *drawingAPI)
      : m_x(x), m_y(y), m_radius(radius), m_drawingAPI(drawingAPI) {}
  void draw() { m_drawingAPI->drawCircle(m_x, m_y, m_radius); }
  void resizeByPercentage(double pct) { m_radius *= pct; }

private:
  double m_x, m_y, m_radius;
  DrawingAPI *m_drawingAPI;
};

int main(void) {
  CircleShape circle1(1, 2, 3, new DrawingAPI1());
  CircleShape circle2(5, 7, 11, new DrawingAPI2());
  circle1.resizeByPercentage(2.5);
  circle2.resizeByPercentage(2.5);
  circle1.draw();
  circle2.draw();
  return 0;
}
```

### 组合模式

组合使得用户一样独立地看待对象和对象的组合。**组合模式**（Composite Pattern）包含这两种方式。使用这一模式，可以构造出树状的层次结构。

```c++
#include <algorithm> // std::for_each
#include <iostream>  // std::cout
#include <memory>    // std::unique_ptr
#include <vector>
using namespace std;

class Graphic {
public:
  virtual void print() const = 0;
  virtual ~Graphic() {}
};

class Ellipse : public Graphic {
public:
  void print() const { cout << "Ellipse \n"; }
};

class CompositeGraphic : public Graphic {
public:
  void print() const {
    for (Graphic *a : graphicList_) {
      a->print();
    }
  }

  void add(Graphic *aGraphic) { graphicList_.push_back(aGraphic); }

private:
  vector<Graphic *> graphicList_;
};

int main() {
  // Initialize four ellipses
  const unique_ptr<Ellipse> ellipse1(new Ellipse());
  const unique_ptr<Ellipse> ellipse2(new Ellipse());
  const unique_ptr<Ellipse> ellipse3(new Ellipse());
  const unique_ptr<Ellipse> ellipse4(new Ellipse());

  // Initialize three composite graphics
  const unique_ptr<CompositeGraphic> graphic(new CompositeGraphic());
  const unique_ptr<CompositeGraphic> graphic1(new CompositeGraphic());
  const unique_ptr<CompositeGraphic> graphic2(new CompositeGraphic());

  // Composes the graphics
  graphic1->add(ellipse1.get());
  graphic1->add(ellipse2.get());
  graphic1->add(ellipse3.get());

  graphic2->add(ellipse4.get());

  graphic->add(graphic1.get());
  graphic->add(graphic2.get());

  // Prints the complete graphic (four times the string "Ellipse")
  graphic->print();
  return 0;
}
```

### 装饰器模式

**装饰器模式**（Decorator Pattern）帮助可以动态地将额外的行为或责任添加至对象。装饰器为子类扩展功能提供了灵活的替代方案。也被叫做 "Wrapper"。如果你的应用需要做某种过滤，装饰器模式是非常值得考虑的一种模式。

```c++
#include <iostream>
#include <memory>
#include <string>

class Interface {
public:
  virtual ~Interface() {}
  virtual void write(std::string &) = 0;
};

class Core : public Interface {
public:
  ~Core() { std::cout << "Core destructor called.\n"; }
  virtual void write(std::string &text) override{}; // Do nothing.
};

class Decorator : public Interface {
private:
  std::unique_ptr<Interface> interface;

public:
  Decorator(std::unique_ptr<Interface> c) { interface = std::move(c); }
  virtual void write(std::string &text) override { interface->write(text); }
};

class MessengerWithSalutation : public Decorator {
private:
  std::string salutation;

public:
  MessengerWithSalutation(std::unique_ptr<Interface> c, const std::string &str)
      : Decorator(std::move(c)), salutation(str) {}
  ~MessengerWithSalutation() { std::cout << "Messenger destructor called.\n"; }
  virtual void write(std::string &text) override {
    text = salutation + "\n\n" + text;
    Decorator::write(text);
  }
};

class MessengerWithValediction : public Decorator {
private:
  std::string valediction;

public:
  MessengerWithValediction(std::unique_ptr<Interface> c, const std::string &str)
      : Decorator(std::move(c)), valediction(str) {}
  ~MessengerWithValediction() {
    std::cout << "MessengerWithValediction destructor called.\n";
  }
  virtual void write(std::string &text) override {
    Decorator::write(text);
    text += "\n\n" + valediction;
  }
};

int main() {
  const std::string salutation = "Greetings,";
  const std::string valediction = "Sincerly, Andy";
  std::string message1 = "This message is not decorated.";
  std::string message2 = "This message is decorated with a salutation.";
  std::string message3 = "This message is decorated with a valediction.";
  std::string message4 =
      "This message is decorated with a salutation and a valediction.";

  std::unique_ptr<Interface> messenger1 = std::make_unique<Core>();
  std::unique_ptr<Interface> messenger2 =
      std::make_unique<MessengerWithSalutation>(std::make_unique<Core>(),
                                                salutation);
  std::unique_ptr<Interface> messenger3 =
      std::make_unique<MessengerWithValediction>(std::make_unique<Core>(),
                                                 valediction);
  std::unique_ptr<Interface> messenger4 =
      std::make_unique<MessengerWithValediction>(
          std::make_unique<MessengerWithSalutation>(std::make_unique<Core>(),
                                                    salutation),
          valediction);

  messenger1->write(message1);
  std::cout << message1 << '\n';
  std::cout << "\n------------------------------\n\n";

  messenger2->write(message2);
  std::cout << message2 << '\n';
  std::cout << "\n------------------------------\n\n";

  messenger3->write(message3);
  std::cout << message3 << '\n';
  std::cout << "\n------------------------------\n\n";

  messenger4->write(message4);
  std::cout << message4 << '\n';
  std::cout << "\n------------------------------\n\n";
}
```

以上代码的输出为

```
This message is not decorated.

------------------------------

Greetings,

This message is decorated with a salutation.

------------------------------

This message is decorated with a valediction.

Sincerly, Andy

------------------------------

Greetings,

This message is decorated with a salutation and a valediction.

Sincerly, Andy

------------------------------

MessengerWithValediction destructor called.
Messenger destructor called.
Core destructor called.
MessengerWithValediction destructor called.
Core destructor called.
Messenger destructor called.
Core destructor called.
Core destructor called.
```

### 外观模式

**外观模式**（Facade Pattern）通过向客户提供接口来隐藏系统的复杂性，客户可以从这一集成的接口上访问整个系统。外观模式定义了更高抽象层次的接口来提高子系统的易用性。比如通过调用其他几个类让一个类执行复杂的过程。

```c++
/*
 * 外观模式是我认为最简单的设计模式之一……以下是一个简单的示例。
 *
 * 假设你的新家是个智能房屋，所有的家居都可被远程控制。这样向要打开灯只需要按下按钮
 * - 其他的设备比如电视、交流电、闹钟、音乐等一样。
 *
 * 当你离开家时，你需要按无数个按钮来确定所有的东西都已经关闭才能出门，如果你像我一
 * 样懒就会觉得太烦了，所以我使用定义了个“外观”在出门和回家时使用（“外观”函数代表那
 * 些按钮）所以当我回家和出门的时候，只需要按一个按钮，它就能帮我搞定所有这些问题。
 */

#include <iostream>
#include <string>

using namespace std;

class Alarm {
public:
  void alarmOn() { cout << "Alarm is on and house is secured" << endl; }
  void alarmOff() {
    cout << "Alarm is off and you can go into the house" << endl;
  }
};

class Ac {
public:
  void acOn() { cout << "Ac is on" << endl; }
  void acOff() { cout << "AC is off" << endl; }
};

class Tv {
public:
  void tvOn() { cout << "Tv is on" << endl; }
  void tvOff() { cout << "TV is off" << endl; }
};

class HouseFacade {
  Alarm alarm;
  Ac ac;
  Tv tv;

public:
  HouseFacade() {}

  void goToWork() {
    ac.acOff();
    tv.tvOff();
    alarm.alarmOn();
  }

  void comeHome() {
    alarm.alarmOff();
    ac.acOn();
    tv.tvOn();
  }
};

int main() {
  HouseFacade hf;

  // 现在不需要按那么一系列按钮来打开或关闭这些功能，多亏了外观模式我现在只需要
  // 两个函数就能搞定
  hf.goToWork();
  hf.comeHome();
}
```

以上程序的输出为：

```
AC is off
TV is off
Alarm is on and house is secured
Alarm is off and you can go into the house
Ac is on
Tv is on
```

### 享元模式

**享元模式**（Flyweight Pattern）通过在对象间共享属性来节约内存。假设有大量相似的对象，大部分他们的属性都一样。很自然地，可以把这些相同的属性移至另外一个数据结构，给每个对象都提供一个指向该数据结构的指针。

```c++
#include <iostream>
#include <string>
#include <vector>

#define NUMBER_OF_SAME_TYPE_CHARS 3;

// 定义享元对象类
class FlyweightCharacter;

/*
 * FlyweightCharacterAbstractBuilder
 * 是一个包含很多对象共享属性的类。这样我们可以将这些属性独立出来，让原先的类
 * 轻量化。更多细节请看 main 函数中的注释。
 */
class FlyweightCharacterAbstractBuilder {
  FlyweightCharacterAbstractBuilder() {}
  ~FlyweightCharacterAbstractBuilder() {}

public:
  static std::vector<float> fontSizes;
  static std::vector<std::string> fontNames;  // 假设每个字符串 6 个字节

  static void setFontsAndNames();
  static FlyweightCharacter
  createFlyweightCharacter(unsigned short fontSizeIndex,
                           unsigned short fontNameIndex,
                           unsigned short positionInStream);
};

std::vector<float> FlyweightCharacterAbstractBuilder::fontSizes(3);
std::vector<std::string> FlyweightCharacterAbstractBuilder::fontNames(3);
void FlyweightCharacterAbstractBuilder::setFontsAndNames() {
  fontSizes[0] = 1.0;
  fontSizes[1] = 1.5;
  fontSizes[2] = 2.0;

  fontNames[0] = "first_font";
  fontNames[1] = "second_font";
  fontNames[2] = "third_font";
}

class FlyweightCharacter {
  unsigned short fontSizeIndex; // 序号而非真正的字体大小
  unsigned short fontNameIndex; // 序号而非真正的字体名字

  unsigned positionInStream;

public:
  FlyweightCharacter(unsigned short fontSizeIndex, unsigned short fontNameIndex,
                     unsigned short positionInStream)
      : fontSizeIndex(fontSizeIndex), fontNameIndex(fontNameIndex),
        positionInStream(positionInStream) {}
  void print() {
    std::cout << "Font Size: "
              << FlyweightCharacterAbstractBuilder::fontSizes[fontSizeIndex]
              << ", font Name: "
              << FlyweightCharacterAbstractBuilder::fontNames[fontNameIndex]
              << ", character stream position: " << positionInStream
              << std::endl;
  }
  ~FlyweightCharacter() {}
};

FlyweightCharacter FlyweightCharacterAbstractBuilder::createFlyweightCharacter(
    unsigned short fontSizeIndex, unsigned short fontNameIndex,
    unsigned short positionInStream) {
  FlyweightCharacter fc(fontSizeIndex, fontNameIndex, positionInStream);

  return fc;
}

int main(int argc, char **argv) {
  std::vector<FlyweightCharacter> chars;

  FlyweightCharacterAbstractBuilder::setFontsAndNames();
  unsigned short limit = NUMBER_OF_SAME_TYPE_CHARS;

  for (unsigned short i = 0; i < limit; i++) {
    chars.push_back(
        FlyweightCharacterAbstractBuilder::createFlyweightCharacter(0, 0, i));
    chars.push_back(FlyweightCharacterAbstractBuilder::createFlyweightCharacter(
        1, 1, i + 1 * limit));
    chars.push_back(FlyweightCharacterAbstractBuilder::createFlyweightCharacter(
        2, 2, i + 2 * limit));
  }
  /*
   * 每个字符类中链接至它的字体名称和大小，这样我们得到：
   *
   * 每个对象为 fontNameIndex 和 fontSizeIndex 各占 2 个字节，否则，float 需要 4
   * 字节，string 需要 6 个字节。这样，每个对象可以节省 6 + 4 - 2 - 2 = 6 比特。
   *
   * 这一设计模式将多个对象共享的属性移至另外的容器内。而原对象中不再保存这些数
   * 据，而是保存指向节省内存轻量化对象容器的链接。被另外保存的这些属性的容器，应
   * 该确实能够节省大量的内存，使得对象与之前相比更加轻便。这也就是这一设计模式名
   * 称的来源：flyweight（即非常轻）。
   */
  for (unsigned short i = 0; i < chars.size(); i++) {
    chars[i].print();
  }

  std::cin.get();
  return 0;
}
```

### 代理模式

**代理模式**（Proxy Pattern）为一个对象提供代理或占位符，使另一个对象控制对它的访问。通常用于需要使用简单的对象来代表复杂的。如果某个对象的创建的代价较高，可以将其推迟到非常需要时再创建，同时用一个更简单的对象作为占位符。这一占位符便叫做复杂对象的“代理”。

```c++
#include <iostream>
#include <memory>

class ICar {
public:
  virtual ~ICar() { std::cout << "ICar destructor!" << std::endl; }
  virtual void DriveCar() = 0;
};

class Car : public ICar {
public:
  void DriveCar() override { std::cout << "Car has been driven!" << std::endl; }
};

class ProxyCar : public ICar {
public:
  ProxyCar(int driver_age) : driver_age_(driver_age) {}
  void DriveCar() override {
    if (driver_age_ > 16) {
      real_car_->DriveCar();
    } else {
      std::cout << "Sorry, the driver is too young to drive." << std::endl;
    }
  }

private:
  std::unique_ptr<ICar> real_car_ = std::make_unique<Car>();
  int driver_age_;
};

int main() {
  std::unique_ptr<ICar> car = std::make_unique<ProxyCar>(16);
  car->DriveCar();

  car = std::make_unique<ProxyCar>(25);
  car->DriveCar();
  return 0;
}
```

### 奇异递归模板模式

这一技术（Curiously Recurring Template）更出名的名称是[Mixin](https://en.wikipedia.org/wiki/Mixin)，Mixin 是表达抽象的非常有用的工具。

### 基于接口编程

**基于接口编程**（Interface-based Programming）与模块化编程（Modular Programming）和面向对象编程关系密切，他将应用定义为相互耦合的模块（相互连接并通过接口相互插入）。模块不需要妥协于其他模块的内容便可以拔出、替换或升级。

这使得整个系统的复杂度大幅降低。基于接口编程相比模块化编程添加了更多的内容，因为它坚持将接口添加至这些模块。这样整个系统被视为一系列组件，其中的接口来帮助他们交互。

基于接口编程提高了程序的模块化，从而提高了其后续开发周期中的可维护性，尤其每个模块必须由不同的团队开发时。这一方法已经存在很久了，它是 CORBA 等框架背后的核心技术。

尤其当第三方要为已建立的系统开发另外的组件时，这样的设计模式很方便。他们只需要开发适应于程序供应者提供的接口的组件即可。

这样接口的发布者要确保他不会改变接口，而订阅者也达成完全只开发相应的接口。因此，接口往往又被成为合同协议（Contractual agreement），而基于此的编程实例被称为基于接口编程。

## 行为型模式

### 责任链模式

**责任链模式**（chain of responsibility pattern）通过给多个对象处理请求的机会来避免将请求的发送方与接收方耦合。链接到接收对象，将请求沿着链接传递直至一个对象处理之。

```c++
#include <iostream>
using namespace std;

class Handler {
protected:
  Handler *next;

public:
  Handler() { next = NULL; }
  virtual ~Handler() {}
  virtual void request(int value) = 0;
  void setNextHandler(Handler *nextInLine) { next = nextInLine; }
};

class SpecialHandler : public Handler {
private:
  int myLimit;
  int myId;

public:
  SpecialHandler(int limit, int id) {
    myLimit = limit;
    myId = id;
  }

  ~SpecialHandler() {}

  void request(int value) {
    if (value < myLimit) {
      cout << "Handler " << myId << " handled the request with a limit of "
           << myLimit << endl;
    } else if (next != NULL) {
      next->request(value);
    } else {
      cout << "Sorry, I am the last handler (" << myId
           << ") and I can't handle the request." << endl;
    }
  }
};

int main() {
  Handler *h1 = new SpecialHandler(10, 1);
  Handler *h2 = new SpecialHandler(20, 2);
  Handler *h3 = new SpecialHandler(30, 3);

  h1->setNextHandler(h2);
  h2->setNextHandler(h3);

  h1->request(18);
  h1->request(40);

  delete h1;
  delete h2;
  delete h3;

  return 0;
}
```

### 命令模式

**命令模式**（Command Pattern）是一种对象行为模式，它通过将请求封装为对象来解耦发送者与接受者，从而可以使用不同的请求、队列或日志请求和支持可撤销操作来将客户端参数化。这一模式可以被视为是回调方法在面向对象中的等价方法。

回调（Call Back）：该功能被注册后会根据用户的操作在一段时间后被调用。

```c++
#include <iostream>
using namespace std;

// 命令接口
class Command {
public:
  virtual void execute() = 0;
};

// 接收者类
class Light {
public:
  Light() {}
  void turnOn() { cout << "The light is on" << endl; }
  void turnOff() { cout << "The light is off" << endl; }
};

// 开灯命令
class FlipUpCommand : public Command {
public:
  FlipUpCommand(Light &light) : theLight(light) {}
  virtual void execute() { theLight.turnOn(); }

private:
  Light &theLight;
};

// 关灯命令
class FlipDownCommand : public Command {
public:
  FlipDownCommand(Light &light) : theLight(light) {}
  virtual void execute() { theLight.turnOff(); }

private:
  Light &theLight;
};

class Switch {
public:
  Switch(Command &flipUpCmd, Command &flipDownCmd)
      : flipUpCommand(flipUpCmd), flipDownCommand(flipDownCmd) {}
  void flipUp() { flipUpCommand.execute(); }
  void flipDown() { flipDownCommand.execute(); }

private:
  Command &flipUpCommand;
  Command &flipDownCommand;
};

int main() {
  Light lamp;
  FlipUpCommand switchUp(lamp);
  FlipDownCommand switchDown(lamp);

  Switch s(switchUp, switchDown);
  s.flipUp();
  s.flipDown();
}
```

### 解释器模式

给定一种语言，定义其语法的形式，使用此形式来解释语言的句子的解释器，即为**解释器模式**（Interpreter Pattern）。

TODO：建议读者参考 lex 和 yacc，或它的衍生如 flex 和 bison 作为解决该问题的替代品。

```c++
#include <iostream>
#include <list>
#include <map>
#include <string>

namespace wikibooks_design_patterns {

// 这里的实现基于 Java 的示例
typedef std::string String;
struct Expression;
typedef std::map<String, Expression *> Map;
typedef std::list<Expression *> Stack;

struct Expression {
  virtual int interpret(Map variables) = 0;
  virtual ~Expression() {}
};

class Number : public Expression {
private:
  int number;

public:
  Number(int number) { this->number = number; }
  int interpret(Map variables) { return number; }
};

class Plus : public Expression {
  Expression *leftOperand;
  Expression *rightOperand;

public:
  Plus(Expression *left, Expression *right) {
    leftOperand = left;
    rightOperand = right;
  }
  ~Plus() {
    delete leftOperand;
    delete rightOperand;
  }

  int interpret(Map variables) {
    return leftOperand->interpret(variables) +
           rightOperand->interpret(variables);
  }
};

class Minus : public Expression {
  Expression *leftOperand;
  Expression *rightOperand;

public:
  Minus(Expression *left, Expression *right) {
    leftOperand = left;
    rightOperand = right;
  }
  ~Minus() {
    delete leftOperand;
    delete rightOperand;
  }

  int interpret(Map variables) {
    return leftOperand->interpret(variables) -
           rightOperand->interpret(variables);
  }
};

class Variable : public Expression {
  String name;

public:
  Variable(String name) { this->name = name; }
  int interpret(Map variables) {
    if (variables.end() == variables.find(name))
      return 0;
    return variables[name]->interpret(variables);
  }
};

// 解释器模式不进行解析器，为完整性而提供该解析器。
class Evaluator : public Expression {
  Expression *syntaxTree;

public:
  Evaluator(String expression) {
    Stack expressionStack;

    size_t last = 0;
    for (size_t next = 0; String::npos != last;
         last = (String::npos == next) ? next : (1 + next)) {
      next = expression.find(' ', last);
      String token(expression.substr(last, (String::npos == next)
                                               ? (expression.length() - last)
                                               : (next - last)));

      if (token == "+") {
        Expression *right = expressionStack.back();
        expressionStack.pop_back();
        Expression *left = expressionStack.back();
        expressionStack.pop_back();
        Expression *subExpression = new Plus(right, left);
        expressionStack.push_back(subExpression);
      } else if (token == "-") {
        // 从栈中移除首个和末个操作符很有必要
        Expression *right = expressionStack.back();
        expressionStack.pop_back();
        Expression *left = expressionStack.back();
        expressionStack.pop_back();
        Expression *subExpression = new Minus(left, right);
        expressionStack.push_back(subExpression);
      } else
        expressionStack.push_back(new Variable(token));
    }

    syntaxTree = expressionStack.back();
    expressionStack.pop_back();
  }

  ~Evaluator() { delete syntaxTree; }

  int interpret(Map context) { return syntaxTree->interpret(context); }
};

} // namespace wikibooks_design_patterns

int main() {
  using namespace wikibooks_design_patterns;

  Evaluator sentence("w x z - +");

  static const int sequences[][3] = {
      {5, 10, 42},
      {1, 3, 2},
      {7, 9, -5},
  };
  for (size_t i = 0; sizeof(sequences) / sizeof(sequences[0]) > i; ++i) {
    Map variables;
    variables["w"] = new Number(sequences[i][0]);
    variables["x"] = new Number(sequences[i][1]);
    variables["z"] = new Number(sequences[i][2]);
    int result = sentence.interpret(variables);
    for (Map::iterator it = variables.begin(); variables.end() != it; ++it)
      delete it->second;
    std::cout << "Interpreter result: " << result << std::endl;
  }
}
```

### 迭代器模式

**迭代器模式**（Iterator Pattern）通常被用于 STL 来遍历各种容器。充分理解这一模式可以将开发者解放出来，以创建高度可冲用和易理解的数据容器。

迭代器的基本想法是它允许对容器进行迭代（像一个沿着数组移动的指针）。然而，要获得容器的下一个元素，你不需要知道容器是如何构造的。这正是迭代器的工作所在。通过简单地使用迭代器提供的成员函数，你可以按照容器的预订顺序从第一个到最后一个进行遍历。

我们首先考虑一维数组，使用一个指针从头至尾进行移动。这一示例假定你已掌握指针的相关知识。注意，这里的 `it` 和 `itr`　是迭代器（iterator）的简写。

```c++
int main() {
  const int ARRAY_LEN = 42;
  int *myArray = new int[ARRAY_LEN];
  // 将迭代器指向数组中的第一个内存位置
  int *arrayItr = myArray;
  // i 将迭代器沿数组移动，使其为其在数组中的位置
  for (int i = 0; i < ARRAY_LEN; ++i) {
    // 将 arrayItr 赋值为 i 目前位置的值
    *arrayItr = i;
    // 通过递增指针，我们将其移动至数组的下一个位置。由于指针机制处理的迭
    // 代，这对与内存连续的容器是很容易的
    ++arrayItr;
  }
  // 别觉得烦，自己清理下
  delete[] myArray;
}
```

以上代码对于数组来说效率很高，那我们应该如何遍历一个内存不连续的链表呢？考虑以下简单链表的实现。

```c++
class IteratorCannotMoveToNext {}; // Error class
class MyIntLList {
public:
  // 类 Node 代表链表中的一个元素。这个节点包含指向下一个和上一个的
  // 节点，因此用户可以从一个位置移动到下一个，或回到上一个位置。注
  // 意遍历链表的复杂度为 O(N)，搜索的复杂度相同，由于链表并非排好
  // 序的。
  class Node {
  public:
    Node() : mNextNode(0), mPrevNode(0), mValue(0) {}
    Node *mNextNode;
    Node *mPrevNode;
    int mValue;
  };
  MyIntLList() : mSize(0) {}
  ~MyIntLList() {
    while (!Empty())
      pop_front();
  }
  int Size() const { return mSize; }
  // 将值添加到链表的末尾
  void push_back(int value) {
    Node *newNode = new Node;
    newNode->mValue = value;
    newNode->mPrevNode = mTail;
    mTail->mNextNode = newNode;
    mTail = newNode;
    ++mSize;
  }
  // 将值从链表的开头移除
  void pop_front() {
    if (Empty())
      return;
    Node *tmpnode = mHead;
    mHead = mHead->mNextNode;
    delete tmpnode;
    --mSize;
  }
  bool Empty() { return mSize == 0; }
  // 迭代器应定义于此，我们先完成链表的定义

private:
  Node *mHead;
  Node *mTail;
  int mSize;
};
```

该链表没有连续的内存，因此不能直接使用指针算法实现。而我们不想将其内部暴露给其他的开发者，这样迫使他们学习，我们也不可改变其结构。

迭代器正适用于此。通用的接口使得其他开发者更易了解如何使用容器，且对其他开发者将遍历的相关逻辑隐藏。

以下为迭代器的代码。

```c++
/*
 *  迭代器类了解聊表的间隔，这样才可以移动到下一个元素。在以下实现中，我选
 *  择了最经典的遍历方式即重载自曾操作符。对双向链表更彻底的实现应该包括自
 *  减操作符，这样迭代器可以反向移动。
 */
class Iterator {
public:
  Iterator(Node *position) : mCurrNode(position) {}
  const Iterator &operator++() {
    if (mCurrNode == 0 || mCurrNode->mNextNode == 0)
      throw IteratorCannotMoveToNext();
    e mCurrNode = mCurrNode->mNextNode;
    return *this;
  }
  Iterator operator++(int) {
    Iterator tempItr = *this;
    ++(*this);
    return tempItr;
  }
  // 取值操作符返回当前的节点，再次对它取值即可获得 int 值。
  // TODO：检查重载取值操作符的语法。
  Node *operator*() { return mCurrNode; }
  // TODO：实现方向操作符，并清理以下的示例用法。
private:
  Node *mCurrNode;
};
// 以下的两个函数使得可以对链表类创建迭代器。
// 第一个迭代器应指向容器中的第一个元素。
Iterator Begin() { return Iterator(mHead); }
// 最后一个迭代器应为容器中最后一个元素的下一个。
Iterator End() { return Iterator(0); }
```

有了以上的实现，我们不需要知道容器的大小和它组织数据的方式，我们就可以按顺序遍历每个元素、修改或简单地访问数据。这些只需要 `Begin()` 与 `End()` 函数即可访问。

```c++
// 创建链表
MyIntLList myList;
// 向链表添加一些元素
for (int i = 0; i < 10; ++i)
  myList.push_back(i);
// 遍历链表，每个元素增加 42
for (MyIntLList::Iterator it = myList.Begin(); it != myList.End(); ++it)
  (*it)->mValue += 42;
```

TODO

- 对 STL 中迭代器和其在算法库中作用的讨论。
- 迭代器的最佳实现。
- 创建与使用的注意事项。
- 使用 [] 操作符更易理解。
- 关于模板对生成代码大小的注意事项（这可能可以写一份不错的学生研究论文）。

以下代码给出具有通用模板的迭代器模式的实现：

```c++
/************************************************************************/
/* Iterator.h                                                           */
/************************************************************************/
#ifndef MY_ITERATOR_HEADER
#define MY_ITERATOR_HEADER

#include <iterator>
#include <set>
#include <vector>

//////////////////////////////////////////////////////////////////////////
template <class T, class U> class Iterator {
public:
  typedef typename std::vector<T>::iterator iter_type;
  Iterator(U *pData) : m_pData(pData) { m_it = m_pData->m_data.begin(); }

  void first() { m_it = m_pData->m_data.begin(); }

  void next() { m_it++; }

  bool isDone() { return (m_it == m_pData->m_data.end()); }

  iter_type current() { return m_it; }

private:
  U *m_pData;
  iter_type m_it;
};

template <class T, class U, class A> class setIterator {
public:
  typedef typename std::set<T, U>::iterator iter_type;

  setIterator(A *pData) : m_pData(pData) { m_it = m_pData->m_data.begin(); }

  void first() { m_it = m_pData->m_data.begin(); }

  void next() { m_it++; }

  bool isDone() { return (m_it == m_pData->m_data.end()); }

  iter_type current() { return m_it; }

private:
  A *m_pData;
  iter_type m_it;
};
#endif
```

```c++
/* Aggregate.h                                                          */
/************************************************************************/
#ifndef MY_DATACOLLECTION_HEADER
#define MY_DATACOLLECTION_HEADER
#include "Iterator.h"

template <class T> class aggregate {
  friend class Iterator<T, aggregate>;

public:
  void add(T a) { m_data.push_back(a); }

  Iterator<T, aggregate> *create_iterator() {
    return new Iterator<T, aggregate>(this);
  }

private:
  std::vector<T> m_data;
};
template <class T, class U> class aggregateSet {
  friend class setIterator<T, U, aggregateSet>;

public:
  void add(T a) { m_data.insert(a); }

  setIterator<T, U, aggregateSet> *create_iterator() {
    return new setIterator<T, U, aggregateSet>(this);
  }

  void Print() {
    copy(m_data.begin(), m_data.end(),
         std::ostream_iterator<T>(std::cout, "\n"));
  }

private:
  std::set<T, U> m_data;
};

#endif
```

```c++
/************************************************************************/
/* Iterator Test.cpp                                                    */
/************************************************************************/
#include "Aggregate.h"
#include <iostream>
#include <string>
using namespace std;

class Money {
public:
  Money(int a = 0) : m_data(a) {}

  void SetMoney(int a) { m_data = a; }

  int GetMoney() { return m_data; }

private:
  int m_data;
};

class Name {
public:
  Name(string name) : m_name(name) {}

  const string &GetName() const { return m_name; }

  friend ostream &operator<<(ostream &out, Name name) {
    out << name.GetName();
    return out;
  }

private:
  string m_name;
};

struct NameLess {
  bool operator()(const Name &lhs, const Name &rhs) const {
    return (lhs.GetName() < rhs.GetName());
  }
};

int main() {
  // sample 1
  cout << "________________Iterator with "
          "int______________________________________"
       << endl;
  aggregate<int> agg;

  for (int i = 0; i < 10; i++)
    agg.add(i);

  Iterator<int, aggregate<int>> *it = agg.create_iterator();
  for (it->first(); !it->isDone(); it->next())
    cout << *it->current() << endl;

  // sample 2
  aggregate<Money> agg2;
  Money a(100), b(1000), c(10000);
  agg2.add(a);
  agg2.add(b);
  agg2.add(c);

  cout << "________________Iterator with Class "
          "Money______________________________"
       << endl;
  Iterator<Money, aggregate<Money>> *it2 = agg2.create_iterator();
  for (it2->first(); !it2->isDone(); it2->next())
    cout << it2->current()->GetMoney() << endl;

  // sample 3
  cout << "________________Set Iterator with Class "
          "Name______________________________"
       << endl;

  aggregateSet<Name, NameLess> aset;
  aset.add(Name("Qmt"));
  aset.add(Name("Bmt"));
  aset.add(Name("Cmt"));
  aset.add(Name("Amt"));

  setIterator<Name, NameLess, aggregateSet<Name, NameLess>> *it3 =
      aset.create_iterator();
  for (it3->first(); !it3->isDone(); it3->next())
    cout << (*it3->current()) << endl;
}
```

输出：

```
________________Iterator with int______________________________________
0
1
2
3
4
5
6
7
8
9
________________Iterator with Class Money______________________________
100
1000
10000
________________Set Iterator with Class Name___________________________
Amt
Bmt
Cmt
Qmt
```

### 中介者模式

**中介者模式**（Mediator Pattern）定义封装一组对象交互方式的对象。中介者模式通过避免对象间显式引用来松散耦合，并且使得可以独立地改变交互方式。

```c++
#include <iostream>
#include <list>
#include <string>

class MediatorInterface;

class ColleagueInterface {
  std::string name;

public:
  ColleagueInterface(const std::string &newName) : name(newName) {}
  std::string getName() const { return name; }
  virtual void sendMessage(const MediatorInterface &,
                           const std::string &) const = 0;
  virtual void receiveMessage(const ColleagueInterface *,
                              const std::string &) const = 0;
};

class Colleague : public ColleagueInterface {
public:
  using ColleagueInterface::ColleagueInterface;
  virtual void sendMessage(const MediatorInterface &,
                           const std::string &) const override;

private:
  virtual void receiveMessage(const ColleagueInterface *,
                              const std::string &) const override;
};

class MediatorInterface {
private:
  std::list<ColleagueInterface *> colleagueList;

public:
  const std::list<ColleagueInterface *> &getColleagueList() const {
    return colleagueList;
  }
  virtual void distributeMessage(const ColleagueInterface *,
                                 const std::string &) const = 0;
  virtual void registerColleague(ColleagueInterface *colleague) {
    colleagueList.emplace_back(colleague);
  }
};

class Mediator : public MediatorInterface {
  virtual void distributeMessage(const ColleagueInterface *,
                                 const std::string &) const override;
};

void Colleague::sendMessage(const MediatorInterface &mediator,
                            const std::string &message) const {
  mediator.distributeMessage(this, message);
}

void Colleague::receiveMessage(const ColleagueInterface *sender,
                               const std::string &message) const {
  std::cout << getName() << " received the message from " << sender->getName()
            << ": " << message << std::endl;
}

void Mediator::distributeMessage(const ColleagueInterface *sender,
                                 const std::string &message) const {
  for (const ColleagueInterface *x : getColleagueList())
    if (x != sender) // Do not send the message back to the sender
      x->receiveMessage(sender, message);
}

int main() {
  Colleague *bob = new Colleague("Bob"), *sam = new Colleague("Sam"),
            *frank = new Colleague("Frank"), *tom = new Colleague("Tom");
  Colleague *staff[] = {bob, sam, frank, tom};
  Mediator mediatorStaff, mediatorSamsBuddies;
  for (Colleague *x : staff)
    mediatorStaff.registerColleague(x);
  bob->sendMessage(mediatorStaff, "I'm quitting this job!");
  mediatorSamsBuddies.registerColleague(frank);
  mediatorSamsBuddies.registerColleague(tom); // Sam's buddies only
  sam->sendMessage(mediatorSamsBuddies,
                   "Hooray!  He's gone!  Let's go for a drink, guys!");
  return 0;
}
```

### 备忘录模式

不需要破坏封装，**备忘录模式**（Memento Pattern）捕捉并外化其内部状态，以便之后可以恢复到该状态。尽管四人帮使用友元来实现，但这并不是最好的设计。还可以使用 PIMPL (不透明指针，pointer to implementation or opaque pointer)。最常见的实例为编辑器中的撤销与重做。

发起人（Originator，被保存的对象）创造自身的镜像作为备忘录对象，并将引用传至负责人对象（Caretaker）。负责人对象保存备忘录，直至发起人需要回至备忘录对象中之前的一个状态。

这一模式旧式样例，请查看[备忘](https://perldoc.perl.org/Memoize.html)。

```c++
#include <iostream>
#include <sstream>
#include <string>
#include <vector>

const std::string NAME = "Object";

template <typename T> std::string toString(const T &t) {
  std::stringstream ss;
  ss << t;
  return ss.str();
}

class Memento;

class Object {
private:
  int value;
  std::string name;
  double decimal; // and suppose there are loads of other data members
public:
  Object(int newValue)
      : value(newValue), name(NAME + toString(value)),
        decimal((float)value / 100) {}
  void doubleValue() {
    value = 2 * value;
    name = NAME + toString(value);
    decimal = (float)value / 100;
  }
  void increaseByOne() {
    value++;
    name = NAME + toString(value);
    decimal = (float)value / 100;
  }
  int getValue() const { return value; }
  std::string getName() const { return name; }
  double getDecimal() const { return decimal; }
  Memento *createMemento() const;
  void reinstateMemento(Memento *mem);
};

class Memento {
private:
  Object object;

public:
  Memento(const Object &obj) : object(obj) {}
  Object snapshot() const {
    return object;
  } // want a snapshot of Object itself because of its many data members
};

Memento *Object::createMemento() const { return new Memento(*this); }

void Object::reinstateMemento(Memento *mem) { *this = mem->snapshot(); }

class Command {
private:
  typedef void (Object::*Action)();
  Object *receiver;
  Action action;
  static std::vector<Command *> commandList;
  static std::vector<Memento *> mementoList;
  static int numCommands;
  static int maxCommands;

public:
  Command(Object *newReceiver, Action newAction)
      : receiver(newReceiver), action(newAction) {}
  virtual void execute() {
    if (mementoList.size() < numCommands + 1)
      mementoList.resize(numCommands + 1);
    mementoList[numCommands] =
        receiver->createMemento(); // saves the last value
    if (commandList.size() < numCommands + 1)
      commandList.resize(numCommands + 1);
    commandList[numCommands] = this; // saves the last command
    if (numCommands > maxCommands)
      maxCommands = numCommands;
    numCommands++;
    (receiver->*action)();
  }
  static void undo() {
    if (numCommands == 0) {
      std::cout << "There is nothing to undo at this point." << std::endl;
      return;
    }
    commandList[numCommands - 1]->receiver->reinstateMemento(
        mementoList[numCommands - 1]);
    numCommands--;
  }
  void static redo() {
    if (numCommands > maxCommands) {
      std::cout << "There is nothing to redo at this point." << std::endl;
      return;
    }
    Command *commandRedo = commandList[numCommands];
    (commandRedo->receiver->*(commandRedo->action))();
    numCommands++;
  }
};

std::vector<Command *> Command::commandList;
std::vector<Memento *> Command::mementoList;
int Command::numCommands = 0;
int Command::maxCommands = 0;

int main() {
  int i;
  std::cout << "Please enter an integer: ";
  std::cin >> i;
  Object *object = new Object(i);

  Command *commands[3];
  commands[1] = new Command(object, &Object::doubleValue);
  commands[2] = new Command(object, &Object::increaseByOne);

  std::cout << "0.Exit,  1.Double,  2.Increase by one,  3.Undo,  4.Redo: ";
  std::cin >> i;

  while (i != 0) {
    if (i == 3)
      Command::undo();
    else if (i == 4)
      Command::redo();
    else if (i > 0 && i <= 2)
      commands[i]->execute();
    else {
      std::cout << "Enter a proper choice: ";
      std::cin >> i;
      continue;
    }
    std::cout << "   " << object->getValue() << "  " << object->getName()
              << "  " << object->getDecimal() << std::endl;
    std::cout << "0.Exit,  1.Double,  2.Increase by one,  3.Undo,  4.Redo: ";
    std::cin >> i;
  }
}
```

### 观察者模式

**观察者模式**（Observer Pattern）定义对象间一对多的依赖，这样当一个对象状态改变，所有依赖于它的都会被提醒并自动更新。

#### 问题

在应用的一个或多个地方，我们需要注意系统的事件或状态改变，最好有一种标准的方式来订阅监听系统事件和通知对事件感兴趣的组件。当一个感兴趣的组件订阅系统事件或应用状态改变后，通知应该可以自动匹配。当然，还应该允许其退订。

#### 限制

观察者和可观察者应该以对象的形式表示。观察者对象会被可观察对象通知。

#### 方案

在订阅监听事件后，对象会以函数调用的方式被通知。

```c++
#include <algorithm>
#include <iostream>
#include <list>
using namespace std;

// 抽象观察者类
class ObserverBoardInterface {
public:
  virtual void update(float a, float b, float c) = 0;
};

// DIsplay 抽象接口类
class DisplayBoardInterface {
public:
  virtual void show() = 0;
};

// 抽象 Subject 类
class WeatherDataInterface {
public:
  virtual void registerOb(ObserverBoardInterface *ob) = 0;
  virtual void removeOb(ObserverBoardInterface *ob) = 0;
  virtual void notifyOb() = 0;
};

// Subject 实体
class ParaWeatherData : public WeatherDataInterface {
public:
  void SensorDataChange(float a, float b, float c) {
    m_humidity = a;
    m_temperature = b;
    m_pressure = c;
    notifyOb();
  }
  void registerOb(ObserverBoardInterface *ob) { m_obs.push_back(ob); }
  void removeOb(ObserverBoardInterface *ob) { m_obs.remove(ob); }

protected:
  void notifyOb() {
    list<ObserverBoardInterface *>::iterator pos = m_obs.begin();
    while (pos != m_obs.end()) {
      ((ObserverBoardInterface *)(*pos))
          ->update(m_humidity, m_temperature, m_pressure);
      (dynamic_cast<DisplayBoardInterface *>(*pos))->show();
      ++pos;
    }
  }

private:
  float m_humidity;
  float m_temperature;
  float m_pressure;
  list<ObserverBoardInterface *> m_obs;
};

// 一个观察者实体
class CurrentConditionBoard : public ObserverBoardInterface,
                              public DisplayBoardInterface {
public:
  CurrentConditionBoard(ParaWeatherData &a) : m_data(a) {
    m_data.registerOb(this);
  }
  void show() {
    cout << "_____CurrentConditionBoard_____" << endl;
    cout << "humidity: " << m_h << endl;
    cout << "temperature: " << m_t << endl;
    cout << "pressure: " << m_p << endl;
    cout << "_______________________________" << endl;
  }

  void update(float h, float t, float p) {
    m_h = h;
    m_t = t;
    m_p = p;
  }

private:
  float m_h;
  float m_t;
  float m_p;
  ParaWeatherData &m_data;
};

// 一个观察者实体
class StatisticBoard : public ObserverBoardInterface,
                       public DisplayBoardInterface {
public:
  StatisticBoard(ParaWeatherData &a)
      : m_maxt(-1000), m_mint(1000), m_avet(0), m_count(0), m_data(a) {
    m_data.registerOb(this);
  }

  void show() {
    cout << "________StatisticBoard_________" << endl;
    cout << "lowest  temperature: " << m_mint << endl;
    cout << "highest temperature: " << m_maxt << endl;
    cout << "average temperature: " << m_avet << endl;
    cout << "_______________________________" << endl;
  }

  void update(float h, float t, float p) {
    ++m_count;
    if (t > m_maxt) {
      m_maxt = t;
    }
    if (t < m_mint) {
      m_mint = t;
    }
    m_avet = (m_avet * (m_count - 1) + t) / m_count;
  }

private:
  float m_maxt;
  float m_mint;
  float m_avet;
  int m_count;
  ParaWeatherData &m_data;
};

int main(int argc, char *argv[]) {

  ParaWeatherData *wdata = new ParaWeatherData;
  CurrentConditionBoard *currentB = new CurrentConditionBoard(*wdata);
  StatisticBoard *statisticB = new StatisticBoard(*wdata);

  wdata->SensorDataChange(10.2, 28.2, 1001);
  wdata->SensorDataChange(12, 30.12, 1003);
  wdata->SensorDataChange(10.2, 26, 806);
  wdata->SensorDataChange(10.3, 35.9, 900);

  wdata->removeOb(currentB);
  wdata->SensorDataChange(100, 40, 1900);

  delete statisticB;
  delete currentB;
  delete wdata;

  return 0;
}
```

### 状态模式

**状态模式**（State Pattern）允许对象在它内部状态改变时改变行为。这一对象看起来就像改变了类。

```c++
#include <cstdlib>
#include <ctime>
#include <iostream>
#include <memory>
#include <string>

enum Input { DUCK_DOWN, STAND_UP, JUMP, DIVE };

class Fighter;
class StandingState;
class JumpingState;
class DivingState;

class FighterState {
public:
  static std::shared_ptr<StandingState> standing;
  static std::shared_ptr<DivingState> diving;
  virtual ~FighterState() = default;
  virtual void handleInput(Fighter &, Input) = 0;
  virtual void update(Fighter &) = 0;
};

class DuckingState : public FighterState {
private:
  int chargingTime;
  static const int FullRestTime = 5;

public:
  DuckingState() : chargingTime(0) {}
  virtual void handleInput(Fighter &, Input) override;
  virtual void update(Fighter &) override;
};

class StandingState : public FighterState {
public:
  virtual void handleInput(Fighter &, Input) override;
  virtual void update(Fighter &) override;
};

class JumpingState : public FighterState {
private:
  int jumpingHeight;

public:
  JumpingState() { jumpingHeight = std::rand() % 5 + 1; }
  virtual void handleInput(Fighter &, Input) override;
  virtual void update(Fighter &) override;
};

class DivingState : public FighterState {
public:
  virtual void handleInput(Fighter &, Input) override;
  virtual void update(Fighter &) override;
};

std::shared_ptr<StandingState> FighterState::standing(new StandingState);
std::shared_ptr<DivingState> FighterState::diving(new DivingState);

class Fighter {
private:
  std::string name;
  std::shared_ptr<FighterState> state;
  int fatigueLevel = std::rand() % 10;

public:
  Fighter(const std::string &newName)
      : name(newName), state(FighterState::standing) {}
  std::string getName() const { return name; }
  int getFatigueLevel() const { return fatigueLevel; }
  virtual void handleInput(Input input) {
    state->handleInput(*this, input);
  } // delegate input handling to 'state'.
  void changeState(std::shared_ptr<FighterState> newState) {
    state = newState;
    updateWithNewState();
  }
  void standsUp() { std::cout << getName() << " stands up." << std::endl; }
  void ducksDown() { std::cout << getName() << " ducks down." << std::endl; }
  void jumps() {
    std::cout << getName() << " jumps into the air." << std::endl;
  }
  void dives() {
    std::cout << getName() << " makes a dive attack in the middle of the jump!"
              << std::endl;
  }

  void feelsStrong() {
    std::cout << getName() << " feels strong!" << std::endl;
  }
  void changeFatigueLevelBy(int change) {
    fatigueLevel += change;
    std::cout << "fatigueLevel = " << fatigueLevel << std::endl;
  }

private:
  virtual void updateWithNewState() {
    state->update(*this);
  } // delegate updating to 'state'
};

void StandingState::handleInput(Fighter &fighter, Input input) {
  switch (input) {
  case STAND_UP:
    std::cout << fighter.getName() << " remains standing." << std::endl;
    return;
  case DUCK_DOWN:
    fighter.changeState(std::shared_ptr<DuckingState>(new DuckingState));
    return fighter.ducksDown();
  case JUMP:
    fighter.jumps();
    return fighter.changeState(std::shared_ptr<JumpingState>(new JumpingState));
  default:
    std::cout << "One cannot do that while standing.  " << fighter.getName()
              << " remains standing by default." << std::endl;
  }
}

void StandingState::update(Fighter &fighter) {
  if (fighter.getFatigueLevel() > 0)
    fighter.changeFatigueLevelBy(-1);
}

void DuckingState::handleInput(Fighter &fighter, Input input) {
  switch (input) {
  case STAND_UP:
    fighter.changeState(FighterState::standing);
    return fighter.standsUp();
  case DUCK_DOWN:
    std::cout << fighter.getName() << " remains in ducking position, ";
    if (chargingTime < FullRestTime)
      std::cout << "recovering in the meantime." << std::endl;
    else
      std::cout << "fully recovered." << std::endl;
    return update(fighter);
  default:
    std::cout << "One cannot do that while ducking.  " << fighter.getName()
              << " remains in ducking position by default." << std::endl;
    update(fighter);
  }
}

void DuckingState::update(Fighter &fighter) {
  chargingTime++;
  std::cout << "Charging time = " << chargingTime << "." << std::endl;
  if (fighter.getFatigueLevel() > 0)
    fighter.changeFatigueLevelBy(-1);
  if (chargingTime >= FullRestTime && fighter.getFatigueLevel() <= 3)
    fighter.feelsStrong();
}

void JumpingState::handleInput(Fighter &fighter, Input input) {
  switch (input) {
  case DIVE:
    fighter.changeState(FighterState::diving);
    return fighter.dives();
  default:
    std::cout << "One cannot do that in the middle of a jump.  "
              << fighter.getName()
              << " lands from his jump and is now standing again." << std::endl;
    fighter.changeState(FighterState::standing);
  }
}

void JumpingState::update(Fighter &fighter) {
  std::cout << fighter.getName() << " has jumped " << jumpingHeight
            << " feet into the air." << std::endl;
  if (jumpingHeight >= 3)
    fighter.changeFatigueLevelBy(1);
}

void DivingState::handleInput(Fighter &fighter, Input) {
  std::cout << "Regardless of what the user input is, " << fighter.getName()
            << " lands from his dive and is now standing again." << std::endl;
  fighter.changeState(FighterState::standing);
}

void DivingState::update(Fighter &fighter) { fighter.changeFatigueLevelBy(2); }

int main() {
  std::srand(std::time(nullptr));
  Fighter rex("Rex the Fighter"), borg("Borg the Fighter");
  std::cout << rex.getName() << " and " << borg.getName()
            << " are currently standing." << std::endl;
  int choice;
  auto chooseAction = [&choice](Fighter &fighter) {
    std::cout << std::endl
              << DUCK_DOWN + 1 << ") Duck down  " << STAND_UP + 1
              << ") Stand up  " << JUMP + 1 << ") Jump  " << DIVE + 1
              << ") Dive in the middle of a jump" << std::endl;
    std::cout << "Choice for " << fighter.getName() << "? ";
    std::cin >> choice;
    const Input input1 = static_cast<Input>(choice - 1);
    fighter.handleInput(input1);
  };
  while (true) {
    chooseAction(rex);
    chooseAction(borg);
  }
}
```

### 策略模式

定义一族算法，逐个封装，然后使其可互换。策略模式使得算法可以根据使用的用户来独立地改变。

```c++
#include <iostream>
using namespace std;

class StrategyInterface {
public:
  virtual void execute() const = 0;
};

class ConcreteStrategyA : public StrategyInterface {
public:
  void execute() const override {
    cout << "Called ConcreteStrategyA execute method" << endl;
  }
};

class ConcreteStrategyB : public StrategyInterface {
public:
  void execute() const override {
    cout << "Called ConcreteStrategyB execute method" << endl;
  }
};

class ConcreteStrategyC : public StrategyInterface {
public:
  void execute() const override {
    cout << "Called ConcreteStrategyC execute method" << endl;
  }
};

class Context {
private:
  StrategyInterface *strategy_;

public:
  explicit Context(StrategyInterface *strategy) : strategy_(strategy) {}

  void set_strategy(StrategyInterface *strategy) { strategy_ = strategy; }

  void execute() const { strategy_->execute(); }
};

int main(int argc, char *argv[]) {
  ConcreteStrategyA concreteStrategyA;
  ConcreteStrategyB concreteStrategyB;
  ConcreteStrategyC concreteStrategyC;

  Context contextA(&concreteStrategyA);
  Context contextB(&concreteStrategyB);
  Context contextC(&concreteStrategyC);

  contextA.execute(); // output: "Called ConcreteStrategyA execute method"
  contextB.execute(); // output: "Called ConcreteStrategyB execute method"
  contextC.execute(); // output: "Called ConcreteStrategyC execute method"

  contextA.set_strategy(&concreteStrategyB);
  contextA.execute(); // output: "Called ConcreteStrategyB execute method"
  contextA.set_strategy(&concreteStrategyC);
  contextA.execute(); // output: "Called ConcreteStrategyC execute method"

  return 0;
}
```

### 模板模式

通过定义算法在操作中定义算法的骨架，将其中一些步骤推迟到子类中，**模板模式**（Template Method）使得不需要改变算法的结构，即可让子类重新定义算法中的某些步骤。

```c++
#include <assert.h>
#include <iostream>

namespace wikibooks_design_patterns {
/*
 * 对数个玩家对抗游戏通用的抽象类，同一时间只能玩一个游戏。
 */

class Game {
public:
  Game() : playersCount(0), movesCount(0), playerWon(-1) {
    srand((unsigned)time(NULL));
  }

  /* A template method : */
  void playOneGame(const int playersCount = 0) {
    if (playersCount) {
      this->playersCount = playersCount;
    }

    InitializeGame();
    assert(this->playersCount);

    int j = 0;
    while (!endOfGame()) {
      makePlay(j);
      j = (j + 1) % this->playersCount;
      if (!j) {
        ++movesCount;
      }
    }
    printWinner();
  }

protected:
  virtual void initializeGame() = 0;
  virtual void makePlay(int player) = 0;
  virtual bool endOfGame() = 0;
  virtual void printWinner() = 0;

private:
  void InitializeGame() {
    movesCount = 0;
    playerWon = -1;

    initializeGame();
  }

protected:
  int playersCount;
  int movesCount;
  int playerWon;
};

// 现在我们扩展这一类来实现具体的游戏
class Monopoly : public Game {

  // 必要实体方法的实现
  void initializeGame() {
    // 初始化玩家
    playersCount = rand() * 7 / RAND_MAX + 2;
    // 初始化金钱
  }
  void makePlay(int player) {
    // 一个回合
    // 决定赢家
    if (movesCount < 20)
      return;
    const int chances = (movesCount > 199) ? 199 : movesCount;
    const int random = MOVES_WIN_CORRECTION * rand() * 200 / RAND_MAX;
    if (random < chances)
      playerWon = player;
  }
  bool endOfGame() {
    // 根据游戏规则返回 true 如果游戏结束
    return (-1 != playerWon);
  }
  void printWinner() {
    assert(playerWon >= 0);
    assert(playerWon < playersCount);

    // 显示赢家
    std::cout << "Monopoly, player " << playerWon << " won in " << movesCount
              << " moves." << std::endl;
  }

private:
  enum {
    MOVES_WIN_CORRECTION = 20,
  };
};

class Chess : public Game {

  void initializeGame() {
    playersCount = 2;
    // 将棋放在棋盘上
  }
  void makePlay(int player) {
    assert(player < playersCount);

    // 一个回合
    // 决定赢家
    if (movesCount < 2)
      return;
    const int chances = (movesCount > 99) ? 99 : movesCount;
    const int random = MOVES_WIN_CORRECTION * rand() * 100 / RAND_MAX;
    // std::cout<<random<<" : "<<chances<<std::endl;
    if (random < chances)
      playerWon = player;
  }
  bool endOfGame() {
    return (-1 != playerWon);
  }
  void printWinner() {
    assert(playerWon >= 0);
    assert(playerWon < playersCount);

    // 显示赢家
    std::cout << "Player " << playerWon << " won in " << movesCount << " moves."
              << std::endl;
  }

private:
  enum {
    MOVES_WIN_CORRECTION = 7,
  };
};

} // namespace wikibooks_design_patterns

int main() {
  using namespace wikibooks_design_patterns;

  Game *game = NULL;

  Chess chess;
  game = &chess;
  for (unsigned i = 0; i < 100; ++i)
    game->playOneGame();

  Monopoly monopoly;
  game = &monopoly;
  for (unsigned i = 0; i < 100; ++i)
    game->playOneGame();

  return 0;
}
```

### 访问者模式

**访问者模式**（Visitor Pattern）表示作用于某个对象结构的操作，该设计模式让你定义一个新的操作而不需改变被操作类。

```c++
#include <iostream>
#include <memory>
#include <string>
#include <vector>
using namespace std;

class Wheel;
class Engine;
class Body;
class Car;

// interface to all car 'parts'
// 面向所有车组件的接口
struct CarElementVisitor {
  virtual void visit(Wheel &wheel) const = 0;
  virtual void visit(Engine &engine) const = 0;
  virtual void visit(Body &body) const = 0;

  virtual void visitCar(Car &car) const = 0;
};

// 某个组件的接口
struct CarElement {
  virtual void accept(const CarElementVisitor &visitor) = 0;
};

// 轮胎，四个轮胎名字各异
class Wheel : public CarElement {
public:
  explicit Wheel(const string &name) : name_(name) {}
  const string &getName() const { return name_; }
  void accept(const CarElementVisitor &visitor) { visitor.visit(*this); }

private:
  string name_;
};

class Engine : public CarElement {
public:
  void accept(const CarElementVisitor &visitor) { visitor.visit(*this); }
};

class Body : public CarElement {
public:
  void accept(const CarElementVisitor &visitor) { visitor.visit(*this); }
};

class Car {
public:
  vector<unique_ptr<CarElement>> &getElements() { return elements_; }

  Car() {
    // 假设以下都不会抛出异常
    elements_.push_back(make_unique<Wheel>("front left"));
    elements_.push_back(make_unique<Wheel>("front right"));
    elements_.push_back(make_unique<Wheel>("back left"));
    elements_.push_back(make_unique<Wheel>("back right"));
    elements_.push_back(make_unique<Body>());
    elements_.push_back(make_unique<Engine>());
  }

private:
  vector<unique_ptr<CarElement>> elements_;
};

// PrintVisitor 和 DoVisitor 使用不同的实现，尽管两者的逻辑各异，但 Car
// 类不需要改变
class CarElementPrintVisitor : public CarElementVisitor {
public:
  void visit(Wheel &wheel) const {
    cout << "Visiting " << wheel.getName() << " wheel" << endl;
  }
  void visit(Engine &engine) const { cout << "Visiting engine" << endl; }
  void visit(Body &body) const { cout << "Visiting body" << endl; }
  void visitCar(Car &car) const {
    cout << endl << "Visiting car" << endl;
    vector<unique_ptr<CarElement>> &elems = car.getElements();

    for (auto &it : elems) {
      it->accept(*this);
    }
    cout << "Visited car" << endl;
  }
};

class CarElementDoVisitor : public CarElementVisitor {
public:
  // 添加特定的对对象的操作而不需要修改之
  void visit(Wheel &wheel) const {
    cout << "Kicking my " << wheel.getName() << " wheel" << endl;
  }
  void visit(Engine &engine) const { cout << "Starting my engine" << endl; }
  void visit(Body &body) const { cout << "Moving my body" << endl; }

  void visitCar(Car &car) const {
    cout << endl << "Starting my car" << endl;
    vector<unique_ptr<CarElement>> &elems = car.getElements();
    for (auto &it : elems) {
      it->accept(*this);
    }
    cout << "Stopped car" << endl;
  }
};

int main() {
  Car car;
  CarElementPrintVisitor printVisitor;
  CarElementDoVisitor doVisitor;

  printVisitor.visitCar(car);
  doVisitor.visitCar(car);

  return 0;
}
```

### MVC 模式

**MVC 模式**（Model-View-Controller 模式）经常被用于需要维护对同一对象不同视图的程序。直到最近，MVC 模式对于尤其是 GUI 编程是非常通用的模式，它将代码分为三个部门。模型，视图和控制器。

模型（Model）代表真正的数据（如数组和链表），或代表数据库的对象。视图（View）是读取模型的接口或客户 GUI。控制器（Controller）提供修改数据的接口，然后选择下一个最好的视图（Next Best View, NBV）。

新手可能觉得这一模式比较冗余，主要因为在运行时需要大量额外的对象，看起来非常的庞大。然而 MVC 模式的理念不是写代码，而是维护，这使得后序维护不需要太多的改变代码。同时，要记住，不同的开发者有自己的强项和弱项，所以团队合作使用 MVC 会更简单。一个视图团队负责于庞大的视图，一个模型团队非常了解数据，一个控制器团队清楚应用流程、处理请求、与模型交互、选择对客户来说最合适的下一个视图。

