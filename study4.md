### 1.创建型模式
创建型模式提供了创建对象的机制， 能够提升已有代码的灵活性和可复用性。

#### 1.1 单例模式
##### 1.1.1 意图
1. 保证一个类只有一个实例。
2. 为该实例提供一个全局访问节点。
##### 1.1.2 实现
所有单例的实现都包含以下两个相同的步骤：

- 将**默认构造函数设为私有**，防止其他对象使用单例类的 new运算符。
- **新建一个静态构建方法作为构造函数**。 该函数会 “偷偷” 调用私有构造函数来创建对象， 并将其保存在一个静态成员变量中。 此后所有对于该函数的调用都将返回这一缓存对象。
![alt](http://image.huawei.com/tiny-lts/v1/images/hi3ms/dd511f6074e3edfdc42c676b30b85b53_645x435.png)



##### 1.1.3 代码实现

```java
// 数据库类会对`getInstance（获取实例）`方法进行定义以让客户端在程序各处都能访问相同的数据库连接实例。
class Database is
    // 保存单例实例的成员变量必须被声明为静态类型。
    private static field instance: Database

    // 单例的构造函数必须永远是私有类型，以防止使用`new`运算符直接调用构造方法。
    private constructor Database() is
        // 部分初始化代码（例如到数据库服务器的实际连接）。
        // ……

    // 用于控制对单例实例的访问权限的静态方法。
    public static method getInstance() is
        if (Database.instance == null) then
            acquireThreadLock() and then
                // 确保在该线程等待解锁时，其他线程没有初始化该实例。
                if (Database.instance == null) then
                    Database.instance = new Database()
        return Database.instance

    // 最后，任何单例都必须定义一些可在其实例上执行的业务逻辑。
    public method query(sql) is
        // 比如应用的所有数据库查询请求都需要通过该方法进行。因此，你可以在这里添加限流或缓冲逻辑。
        // ……

class Application is
    method main() is
        Database foo = Database.getInstance()
        foo.query("SELECT ……")
        // ……
        Database bar = Database.getInstance()
        bar.query("SELECT ……")
        // 变量 `bar` 和 `foo` 中将包含同一个对象。

```
##### 1.1.4 应用场景
1. 如果程序中的某个类对于所有客户端只有一个可用的实例， 可以使用单例模式。
2. 如果你需要更加严格地控制全局变量， 可以使用单例模式。

##### 1.1.5 最佳实例
对于物理上或逻辑上就只应该存在一个的东西，如配置文件、连接池。
都应该用单例模式实现。

#### 1.2 工厂方法模式
##### 1.2.1 简单工厂模式及优化


先从简单工厂模式说起。非常简单，简单工厂可根据方法的参数来选择对何种产品进行初始化并将其返回。

```java
class UserFactory {
    public static function create($type) {
        switch ($type) {
            case 'user': return new User();
            case 'customer': return new Customer();
            case 'admin': return new Admin();
            default:
                throw new Exception('传递的用户类型错误。');
        }
    }
}
```
显然可以优化：
- 抽取出产品（type），工厂只需要返回抽象产品接口；
- 子类工厂决定实例化对象的类型；
- **工厂最主要的职责并不是创建产品**。 一般来说， 创建者类包含一些与产品相关的核心业务逻辑。 工厂方法将这些逻辑处理从具体产品类中分离出来。 

简单说工厂方法模式做了2件事：
- 把所有具体产品都抽象成接口；
- 分离产品的创建和产品相关的核心业务逻辑。
##### 1.2.2 实现
![alt](http://image.huawei.com/tiny-lts/v1/images/hi3ms/432d68d7557b3b41caa0e3f18e7d01e8_990x570.png)


##### 1.2.3 代码实现

```java
// 创建者类声明的工厂方法必须返回一个产品类的对象。创建者的子类通常会提供该方法的实现。
class Dialog is
    // 创建者还可提供一些工厂方法的默认实现。
    abstract method createButton():Button

    // 请注意，创建者的主要职责并非是创建产品。其中通常会包含一些核心业务
    // 逻辑，这些逻辑依赖于由工厂方法返回的产品对象。子类可通过重写工厂方
    // 法并使其返回不同类型的产品来间接修改业务逻辑。
    method render() is
        // 调用工厂方法创建一个产品对象。
        Button okButton = createButton()
        // 现在使用产品。
        okButton.onClick(closeDialog)
        okButton.render()


// 具体创建者将重写工厂方法以改变其所返回的产品类型。
class WindowsDialog extends Dialog is
    method createButton():Button is
        return new WindowsButton()

class WebDialog extends Dialog is
    method createButton():Button is
        return new HTMLButton()


// 产品接口中将声明所有具体产品都必须实现的操作。
interface Button is
    method render()
    method onClick(f)

// 具体产品需提供产品接口的各种实现。
class WindowsButton implements Button is
    method render(a, b) is
        // 根据 Windows 样式渲染按钮。
    method onClick(f) is
        // 绑定本地操作系统点击事件。

class HTMLButton implements Button is
    method render(a, b) is
        // 返回一个按钮的 HTML 表述。
    method onClick(f) is
        // 绑定网络浏览器的点击事件。


class Application is
    field dialog: Dialog

    // 程序根据当前配置或环境设定选择创建者的类型。
    method initialize() is
        config = readApplicationConfigFile()

        if (config.OS == "Windows") then
            dialog = new WindowsDialog()
        else if (config.OS == "Web") then
            dialog = new WebDialog()
        else
            throw new Exception("错误！未知的操作系统。")

    // 当前客户端代码会与具体创建者的实例进行交互，但是必须通过其基本接口
    // 进行。只要客户端通过基本接口与创建者进行交互，你就可将任何创建者子
    // 类传递给客户端。
    method main() is
        this.initialize()
        dialog.render()

```
##### 1.2.4 注意点
如何判断创建者的类型，是程序根据配置或环境自动选择的，与客户端代码无关。客户端不需要区分各个工厂。
##### 1.2.5 应用场景
- 如果无法预知对象确切类别及其依赖关系时， 可使用工厂方法。
- 希望用户能扩展你软件库或框架的内部组件， 可使用工厂方法。
- **如果你希望复用现有对象来节省系统资源， 而不是每次都重新创建对象， 可使用工厂方法**。

在处理大型资源密集型对象 （比如数据库连接） 时，引入连接池。
当业务代码调用 工厂方法时，可以检查连接池是否有空闲连接，有直接**返回空闲连接对象**，没有查看是否达到最大连接数，未达到**直接创建新连接对象并返回**，已达到则等待。

#### 1.3 抽象工厂模式
##### 1.3.1 区别
抽象工厂模式能创建**一系列相关或相互依赖的对象**，而无需指定其具体类。重点就在于系列对象。

什么是 “系列对象”？ 例如有这样一组的对象： 运输工具+ 引擎+ 控制器 。 它可能会有几个变体：

- 汽车+ 内燃机+ 方向盘
- 飞机+ 喷气式发动机+ 操纵杆
##### 1.3.2 实现
![alt](http://image.huawei.com/tiny-lts/v1/images/hi3ms/83257e19d50b9b07877c29591de0e3c2_1080x675.png)


##### 1.3.3 代码实现


```java
// 抽象工厂接口声明了一组能返回不同抽象产品的方法。这些产品属于同一个系列
// 且在高层主题或概念上具有相关性。同系列的产品通常能相互搭配使用。系列产
// 品可有多个变体，但不同变体的产品不能搭配使用。
interface GUIFactory is
    method createButton():Button
    method createCheckbox():Checkbox


// 具体工厂可生成属于同一变体的系列产品。工厂会确保其创建的产品能相互搭配
// 使用。具体工厂方法签名会返回一个抽象产品，但在方法内部则会对具体产品进
// 行实例化。
class WinFactory implements GUIFactory is
    method createButton():Button is
        return new WinButton()
    method createCheckbox():Checkbox is
        return new WinCheckbox()

// 每个具体工厂中都会包含一个相应的产品变体。
class MacFactory implements GUIFactory is
    method createButton():Button is
        return new MacButton()
    method createCheckbox():Checkbox is
        return new MacCheckbox()


// 系列产品中的特定产品必须有一个基础接口。所有产品变体都必须实现这个接口。
interface Button is
    method paint()

// 具体产品由相应的具体工厂创建。
class WinButton implements Button is
    method paint() is
        // 根据 Windows 样式渲染按钮。

class MacButton implements Button is
    method paint() is
        // 根据 macOS 样式渲染按钮

// 这是另一个产品的基础接口。所有产品都可以互动，但是只有相同具体变体的产
// 品之间才能够正确地进行交互。
interface Checkbox is
    method paint()

class WinCheckbox implements Checkbox is
    method paint() is
        // 根据 Windows 样式渲染复选框。

class MacCheckbox implements Checkbox is
    method paint() is
        // 根据 macOS 样式渲染复选框。

// 客户端代码仅通过抽象类型（GUIFactory、Button 和 Checkbox）使用工厂
// 和产品。这让你无需修改任何工厂或产品子类就能将其传递给客户端代码。
class Application is
    private field factory: GUIFactory
    private field button: Button
    constructor Application(factory: GUIFactory) is
        this.factory = factory
    method createUI() is
        this.button = factory.createButton()
    method paint() is
        button.paint()


// 程序会根据当前配置或环境设定选择工厂类型，并在运行时创建工厂（通常在初
// 始化阶段）。
class ApplicationConfigurator is
    method main() is
        config = readApplicationConfigFile()

        if (config.OS == "Windows") then
            factory = new WinFactory()
        else if (config.OS == "Mac") then
            factory = new MacFactory()
        else
            throw new Exception("错误！未知的操作系统。")

        Application app = new Application(factory)

```
##### 1.3.4 应用场景
如果代码需要与多个不同系列的相关产品交互， 但是由于无法提前获取相关信息， 或者出于对未来扩展性的考虑， 你不希望代码基于产品的具体类进行构建， 在这种情况下， 你可以使用抽象工厂。

##### 1.3.5 工厂模式最佳实例
JDBC中的Connection 接口。
模式角色对应：

- 抽象工厂：java.sql.Connection
- 具体工厂：各数据库驱动提供的 Connection 实现类
- 产品族：Statement、PreparedStatement、CallableStatement
- 具体产品：MysqlStatement、MysqlPreparedStatement、PostgresqlStatement 等

具体实现：

```java
// 获取具体工厂（Connection 对象）
Connection conn = DriverManager.getConnection("jdbc:mysql://localhost:3306/db");

// 同一工厂可生产同一产品族的不同产品
Statement stmt = conn.createStatement();// 普通 SQL 语句
PreparedStatement pstmt = conn.prepareStatement(sql); // 预编译语句
CallableStatement cstmt = conn.prepareCall("{call proc}"); // 存储过程调用

```
这里并没有让具体工厂去继承抽象工厂实现。而是用到了Java经典的**服务提供者框架**。
每个数据库驱动必须提供 java.sql.Driver 接口的实现类，并在该实现类的静态初始化块中向 DriverManager 注册自己。接下来，**DriverManager只需要根据url参数，遍历所有已注册的 Driver**，即可判断是哪个具体工厂。

这样应用代码只依赖 Connection 接口，完全不知道具体工厂类名。


#### 1.4 建造者模式
##### 1.4.1 问题和意图
有些基类可能有大量的参数，构造函数怎么写呢？可以创建一个包括所有可能参数的超级构造函数， 并用它来控制对象，**但是**，通常情况下， 绝大部分的参数都没有使用， 这使得对于构造函数的调用十分不简洁。

建造者模式使你能够分步骤创建复杂对象。 该模式允许你使用相同的创建代码生成不同类型和形式的对象。在Java中非常常见，就是builder。

##### 1.4.2 模式结构
![alt](http://image.huawei.com/tiny-lts/v1/images/hi3ms/6fbc1ba2047b5724a4d40ce1efc35dc2_690x870.png)


注意图中的director主管一般不是必须的，builder的灵活性就在于想添加哪个参数就添加哪个，而主管的作用就是定义调用构造步骤的顺序，不灵活。

##### 1.4.3 伪代码实现


```java
// 只有当产品较为复杂且需要详细配置时，使用生成器模式才有意义。下面的两个
// 产品尽管没有同样的接口，但却相互关联。
class Car is
    // 一辆汽车可能配备有 GPS 设备、行车电脑和几个座位。不同型号的汽车（
    // 运动型轿车、SUV 和敞篷车）可能会安装或启用不同的功能。

class Manual is
    // 用户使用手册应该根据汽车配置进行编制，并介绍汽车的所有功能。


// 生成器接口声明了创建产品对象不同部件的方法。
interface Builder is
    method reset()
    method setSeats(……)
    method setEngine(……)
    method setTripComputer(……)
    method setGPS(……)

// 具体生成器类将遵循生成器接口并提供生成步骤的具体实现。你的程序中可能会
// 有多个以不同方式实现的生成器变体。
class CarBuilder implements Builder is
    private field car:Car

    // 一个新的生成器实例必须包含一个在后续组装过程中使用的空产品对象。
    constructor CarBuilder() is
        this.reset()

    // reset（重置）方法可清除正在生成的对象。
    method reset() is
        this.car = new Car()

    // 所有生成步骤都会与同一个产品实例进行交互。
    method setSeats(……) is
        // 设置汽车座位的数量。

    method setEngine(……) is
        // 安装指定的引擎。

    method setTripComputer(……) is
        // 安装行车电脑。

    method setGPS(……) is
        // 安装全球定位系统。

    // 具体生成器需要自行提供获取结果的方法。这是因为不同类型的生成器可能
    // 会创建不遵循相同接口的、完全不同的产品。所以也就无法在生成器接口中
    // 声明这些方法（至少在静态类型的编程语言中是这样的）。
    //
    // 通常在生成器实例将结果返回给客户端后，它们应该做好生成另一个产品的
    // 准备。因此生成器实例通常会在 `getProduct（获取产品）`方法主体末尾
    // 调用重置方法。但是该行为并不是必需的，你也可让生成器等待客户端明确
    // 调用重置方法后再去处理之前的结果。
    method getProduct():Car is
        product = this.car
        this.reset()
        return product

// 生成器与其他创建型模式的不同之处在于：它让你能创建不遵循相同接口的产品。
class CarManualBuilder implements Builder is
    private field manual:Manual

    constructor CarManualBuilder() is
        this.reset()

    method reset() is
        this.manual = new Manual()

    method setSeats(……) is
        // 添加关于汽车座椅功能的文档。

    method setEngine(……) is
        // 添加关于引擎的介绍。

    method setTripComputer(……) is
        // 添加关于行车电脑的介绍。

    method setGPS(……) is
        // 添加关于 GPS 的介绍。

    method getProduct():Manual is
        // 返回使用手册并重置生成器。


// 主管只负责按照特定顺序执行生成步骤。其在根据特定步骤或配置来生成产品时
// 会很有帮助。由于客户端可以直接控制生成器，所以严格意义上来说，主管类并
// 不是必需的。
class Director is
    // 主管可同由客户端代码传递给自身的任何生成器实例进行交互。客户端可通
    // 过这种方式改变最新组装完毕的产品的最终类型。主管可使用同样的生成步
    // 骤创建多个产品变体。
    method constructSportsCar(builder: Builder) is
        builder.reset()
        builder.setSeats(2)
        builder.setEngine(new SportEngine())
        builder.setTripComputer(true)
        builder.setGPS(true)

    method constructSUV(builder: Builder) is
        // ……


// 客户端代码会创建生成器对象并将其传递给主管，然后执行构造过程。最终结果
// 将需要从生成器对象中获取。
class Application is

    method makeCar() is
        director = new Director()

        CarBuilder builder = new CarBuilder()
        director.constructSportsCar(builder)
        Car car = builder.getProduct()

        CarManualBuilder builder = new CarManualBuilder()
        director.constructSportsCar(builder)

        // 最终产品通常需要从生成器对象中获取，因为主管不知晓具体生成器和
        // 产品的存在，也不会对其产生依赖。
        Manual manual = builder.getProduct()

```

注意客户端直接使用具体生成器即可。
##### 1.4.4 应用场景
1. 最常用：避免 “重叠构造函数” 的出现。再也不需要将几十个参数塞进构造函数里了。
2. 希望使用代码创建不同形式的产品 （例如石头或木头房屋） 时， 可使用生成器模式。
##### 1.4.5 现代流式建造者
实际落地时，**不需要builder接口和主管类，而是生成一个内部静态类作为唯一的具体建造者**。
例如，在 Java 领域，Lombok 的 @Builder 注解堪称“零成本”生成器的极致实现。
只要在一个类上标注@Builder注解。
就可以这么使用：

```java
User user = User.builder()
    .name("张三")
    .age(25)
    .email("zhangsan@example.com")
    .build();

```
其中为 User 生成了一个内部静态类 UserBuilder 作为唯一的具体建造者。
且builder() 是一个静态工厂方法，作用是返回一个 UserBuilder 实例，让你可以开始流式地设置参数。
它自动处理了参数校验、默认值，并且强制你在最后调用 build() 来获取一个不可变对象（如果需要）。
##### 1.4.6 最佳实例
经典中的经典，StringBuilder ：

```java
String result = new StringBuilder()
    .append("Hello")
    .append(" ")
    .append("World")
    .append("!")
    .toString();

```
解决的是直接拼接字符串低效且不可读的问题。

#### 1.5 原型模式
##### 1.5.1 问题和意图
如果你有一个对象，希望生成与其完全相同的一个复制品。
原型模式使你能够复制已有对象，而又无需使代码依赖它们所属的类。
##### 1.5.2 应用场景
如果你需要复制一些对象， 同时又希望代码独立于这些对象所属的具体类， 可以使用原型模式。

 这一点考量通常出现在代码需要处理第三方代码通过接口传递过来的对象时。 即使不考虑代码耦合的情况， 你的代码也不能依赖这些对象所属的具体类， 因为你不知道它们的具体信息。

原型模式为客户端代码提供一个通用接口， 客户端代码可通过这一接口与所有实现了克隆的对象进行交互， 它也使得客户端代码与其所克隆的对象具体类独立开来。

##### 1.5.3 最佳实践
1. JavaScript 的原型继承与 Object.create()；
每个 JS 对象都有一个内部原型引用 `[[Prototype]]`，你可以使用 Object.create(proto) 直接以一个对象为“原型”创建新对象。
直接用对象创建对象，还是很神奇的，完美绕开new带来的重复初始化工作。
1. Unity 预制体（Prefab）—— 游戏开发的基石；
预先在编辑器里制作好一个完整的游戏对象，包含模型、脚本、组件、初始数值，这就是“原型”。运行时要大量产生这种对象时，只需调用 Instantiate(prefab)，引擎就会在内存中克隆出一个完全相同的副本，并可后续微调属性。


### 2.结构性模式
这类模式介绍如何将对象和类组装成较大的结构， 并同时保持结构的灵活和高效。
#### 2.1 装饰者模式
##### 2.1.1 问题和意图
将对象放入包含行为的特殊封装对象中来为原对象绑定新的行为，甚至不止一个新的行为。
即增强原对象的能力。
##### 2.1.2 模式结构
![alt](http://image.huawei.com/tiny-lts/v1/images/hi3ms/0571853ec3cfebcc6eee9c2a3807dc9c_720x780.png)

其实Component就是要增强的对象；
装饰者既要实现接口，又要引用这个接口；

##### 2.1.3 伪代码实现

```java
// 装饰可以改变组件接口所定义的操作。
interface DataSource is
    method writeData(data)
    method readData():data

// 具体组件提供操作的默认实现。这些类在程序中可能会有几个变体。
class FileDataSource implements DataSource is
    constructor FileDataSource(filename) { …… }

    method writeData(data) is
        // 将数据写入文件。

    method readData():data is
        // 从文件读取数据。

// 装饰基类和其他组件遵循相同的接口。该类的主要任务是定义所有具体装饰的封
// 装接口。封装的默认实现代码中可能会包含一个保存被封装组件的成员变量，并
// 且负责对其进行初始化。
class DataSourceDecorator implements DataSource is
    protected field wrappee: DataSource

    constructor DataSourceDecorator(source: DataSource) is
        wrappee = source

    // 装饰基类会直接将所有工作分派给被封装组件。具体装饰中则可以新增一些
    // 额外的行为。
    method writeData(data) is
        wrappee.writeData(data)

    // 具体装饰可调用其父类的操作实现，而不是直接调用被封装对象。这种方式
    // 可简化装饰类的扩展工作。
    method readData():data is
        return wrappee.readData()

// 具体装饰必须在被封装对象上调用方法，不过也可以自行在结果中添加一些内容。
// 装饰必须在调用封装对象之前或之后执行额外的行为。
class EncryptionDecorator extends DataSourceDecorator is
    method writeData(data) is
        // 1. 对传递数据进行加密。
        // 2. 将加密后数据传递给被封装对象 writeData（写入数据）方法。

    method readData():data is
        // 1. 通过被封装对象的 readData（读取数据）方法获取数据。
        // 2. 如果数据被加密就尝试解密。
        // 3. 返回结果。

// 你可以将对象封装在多层装饰中。
class CompressionDecorator extends DataSourceDecorator is
    method writeData(data) is
        // 1. 压缩传递数据。
        // 2. 将压缩后数据传递给被封装对象 writeData（写入数据）方法。

    method readData():data is
        // 1. 通过被封装对象的 readData（读取数据）方法获取数据。
        // 2. 如果数据被压缩就尝试解压。
        // 3. 返回结果。


// 选项 1：装饰组件的简单示例
class Application is
    method dumbUsageExample() is
        source = new FileDataSource("somefile.dat")
        source.writeData(salaryRecords)
        // 已将明码数据写入目标文件。

        source = new CompressionDecorator(source)
        source.writeData(salaryRecords)
        // 已将压缩数据写入目标文件。

        source = new EncryptionDecorator(source)
        // 源变量中现在包含：
        // Encryption > Compression > FileDataSource
        source.writeData(salaryRecords)
        // 已将压缩且加密的数据写入目标文件。


// 选项 2：客户端使用外部数据源。SalaryManager（工资管理器）对象并不关心
// 数据如何存储。它们会与提前配置好的数据源进行交互，数据源则是通过程序配
// 置器获取的。
class SalaryManager is
    field source: DataSource

    constructor SalaryManager(source: DataSource) { …… }

    method load() is
        return source.readData()

    method save() is
        source.writeData(salaryRecords)
    // ……其他有用的方法……


// 程序可在运行时根据配置或环境组装不同的装饰堆桟。
class ApplicationConfigurator is
    method configurationExample() is
        source = new FileDataSource("salary.dat")
        if (enabledEncryption)
            source = new EncryptionDecorator(source)
        if (enabledCompression)
            source = new CompressionDecorator(source)

        logger = new SalaryManager(source)
        salary = logger.load()
    // ……

```
##### 2.1.4 应用场景
希望在运行时为对象新增额外的行为， 可以使用装饰模式。
##### 2.1.5 最佳实践
Java I/O 流体系。整个 java.io 包就是装饰者模式的大规模应用。
在 Java I/O 中，InputStream 就是组件接口，FileInputStream 是具体组件，FilterInputStream 是装饰器基类（它持有另一个 InputStream 引用）。

现在想实现一个**带缓冲且能解压的数据输入流**，无需为每一种组合创建子类，分别用BufferedInputStream 和 GZIPInputStream 装饰器来增强。


```java
public class BufferedInputStream extends FilterInputStream {
    private byte[] buffer;
    private int pos, count;
    public BufferedInputStream(InputStream in) { super(in); }
    public int read() throws IOException {
        // 实现带缓冲的读取逻辑
    }
}

public class GZIPInputStream extends FilterInputStream {
    public GZIPInputStream(InputStream in) throws IOException { super(in); }
    public int read() throws IOException {
        // 实现解压读取逻辑
    }
}

```

接下来只需将具体组件依次用所需的装饰器包装即可：

```java
//分别组合
InputStream fileIn = new FileInputStream("data.gz");
InputStream decompressor = new GZIPInputStream(fileIn); // 解压
InputStream bufferedDecompressor = new BufferedInputStream(decompressor); // 加缓冲

// 直接一行链式包装：
InputStream in = new BufferedInputStream(new GZIPInputStream(new FileInputStream("data.gz")));

//接下来这个in再调用read，就会带解压和缓冲
```
#### 2.2 适配器模式
##### 2.2.1 问题和意图
让接口不兼容的对象能够相互合作。
##### 2.2.2 模式结构
![alt](http://image.huawei.com/tiny-lts/v1/images/hi3ms/4b40a7f1ce9809f7c267e260f75de0d5_870x480.png)


目标接口和service（一般不能修改）直接不匹配，用适配器继承目标接口，其中引用service，根据业务需要把它给转换了。
##### 2.2.3 伪代码实现

```java
// 假设你有两个接口相互兼容的类：圆孔（Round­Hole）和圆钉（Round­Peg）。
class RoundHole is
    constructor RoundHole(radius) { …… }

    method getRadius() is
        // 返回孔的半径。

    method fits(peg: RoundPeg) is
        return this.getRadius() >= peg.getRadius()

class RoundPeg is
    constructor RoundPeg(radius) { …… }

    method getRadius() is
        // 返回钉子的半径。


// 但还有一个不兼容的类：方钉（Square­Peg）。
class SquarePeg is
    constructor SquarePeg(width) { …… }

    method getWidth() is
        // 返回方钉的宽度。


// 适配器类让你能够将方钉放入圆孔中。它会对 RoundPeg 类进行扩展，以接收适
// 配器对象作为圆钉。
class SquarePegAdapter extends RoundPeg is
    // 在实际情况中，适配器中会包含一个 SquarePeg 类的实例。
    private field peg: SquarePeg

    constructor SquarePegAdapter(peg: SquarePeg) is
        this.peg = peg

    method getRadius() is
        // 适配器会假扮为一个圆钉，其半径刚好能与适配器实际封装的方钉搭配
        // 起来。
        return peg.getWidth() * Math.sqrt(2) / 2


// 客户端代码中的某个位置。
hole = new RoundHole(5)
rpeg = new RoundPeg(5)
hole.fits(rpeg) // true

small_sqpeg = new SquarePeg(5)
large_sqpeg = new SquarePeg(10)
hole.fits(small_sqpeg) // 此处无法编译（类型不一致）。

small_sqpeg_adapter = new SquarePegAdapter(small_sqpeg)
large_sqpeg_adapter = new SquarePegAdapter(large_sqpeg)
hole.fits(small_sqpeg_adapter) // true
hole.fits(large_sqpeg_adapter) // false

```
##### 2.2.4 应用场景
当你希望使用某个类， 但是其接口与其他代码不兼容时， 可以使用适配器类。
适配器模式允许你创建一个中间层类， 其可作为代码与遗留类、 第三方类或提供怪异接口的类之间的转换器。

##### 2.2.5 最佳实践
我觉得最合适，最有用的场景就是遗留系统整合 —— 新旧接口适配。
有一些老的接口不想修改，但是新的架构与其不适配，此时非常适合用适配器模式。
如创建适配器实现 IUserService，内部持有 OldUserManager 实例并做调用转换：

```java
public class UserServiceAdapter implements IUserService {
    private OldUserManager oldManager;
    
    public User createUser(UserDTO dto) {
        // 适配转换
        String name = dto.getFullName();
        int age = dto.getAge();
        int legacyId = oldManager.addUser(name, age);
        // 将老系统返回的ID封装进新的User对象
        User user = new User();
        user.setId(legacyId);
        user.setName(name);
        return user;
    }
}

```
#### 2.3 外观模式

##### 2.3.1 问题和意图
最好解释的一个，就是**最少知识原则**，为程序库、框架或其他复杂类提供一个简单的接口。
调用者只需要调用外观接口，内部细节不知道，从某方面来说也是封装了所有内部细节。
这个模式理论上来说是最常用的，并且可以和各个模式搭配使用。

##### 2.3.2 模式结构
![alt](http://image.huawei.com/tiny-lts/v1/images/hi3ms/d29cbb3ec2651abd7aa12428c456aed8_840x570.png)

##### 2.3.3 代码实现
以最常见的提交订单为例，对控制器只要提供一个外观即可，而内部库存扣减、生成订单、创建支付单、发送消息通知全在外观内。

```java
public class CheckoutFacade {
    private InventoryService inventory;
    private OrderService order;
    private PaymentService payment;
    private NotificationService notify;

    @Transactional
    public OrderResult placeOrder(OrderRequest req) {
        // 1. 锁定库存
        inventory.reserve(req.getItems());
        try {
            // 2. 创建订单
            Order order = this.order.create(req);
            // 3. 发起支付
            Payment payment = this.payment.charge(order);
            // 4. 发送通知
            notify.sendOrderConfirmation(order);
            return new OrderResult(order, payment);
        } catch (Exception e) {
            // 5. 失败时释放库存（补偿）
            inventory.release(req.getItems());
            throw e;
        }
    }
}

// 前端控制器只调用外观
@PostMapping("/checkout")
public Result checkout(@RequestBody OrderRequest req) {
    return checkoutFacade.placeOrder(req);
}

```
##### 2.3.4 最佳实践

程序员点击 IDE 的“构建并运行”，背后其实是预处理→编译→汇编→链接→打包→部署这一串工具的调用。
IDE 的构建系统本身就是一个巨大的外观，对外只给出“Build”、“Run”、“Debug”等按钮，内部按项目配置调度全部工具链。

这操作就像快捷方式一样。

#### 2.4 代理模式
##### 2.4.1 问题和意图
代理控制着对于原对象的访问，并允许在将请求提交给对象前后进行一些处理。
##### 2.4.2 模式架构
![alt](http://image.huawei.com/tiny-lts/v1/images/hi3ms/f84a1294b4b48a82b22cd9a4e3dde72d_555x555.png)


代理 （Proxy） 类包含一个指向服务对象的引用成员变量。 代理完成其任务 （例如延迟初始化、 记录日志、 访问控制和缓存等） 后会将请求传递给服务对象。
##### 2.4.3 最佳实践
代理模式是绝大多数**中间件**“无感增强”背后的核心机制，是 IoC 与 AOP 的基石。

###### 2.4.3.1 AOP
用代理对象实现非功能需求（日志、事务、权限），调用者拿到的是代理，调用方法时会先执行通知链，再委托目标方法。
业务代码完全与横切逻辑解耦，只需一个 @Transactional 注解即可插入复杂的事务管理。

```java
// 目标接口与实现
public interface OrderService {
    void placeOrder(Order order);
}
@Service
public class OrderServiceImpl implements OrderService {
    @Transactional  // 声明式事务，本质是AOP
    public void placeOrder(Order order) { /* 核心业务 */ }
}

// Spring 生成的代理对象（伪代码）
public class OrderServiceProxy implements OrderService {
    private OrderService target;
    public void placeOrder(Order order) {
        // 1. 前置通知：开启事务
        // 2. 调用目标方法 target.placeOrder(order)
        // 3. 后置/返回通知：提交事务
        // 4. 异常通知：回滚事务
    }
}

```
###### 2.4.3.2 远程服务调用
微服务架构中，服务消费者需要调用远程提供者的接口，但远程调用的网络、序列化、负载均衡等细节极其复杂，不应侵入业务代码。
Dubbo 等 RPC 框架通过 JDK 动态代理为服务接口生成一个远程代理对象。当消费者调用接口方法时，代理拦截调用，将方法名、参数封装成 RPC 请求，通过网络发给服务端，等待响应后再返回结果。
**隐藏网络通信细节**。

```java
// 消费者端代码，只依赖接口
@Reference
private UserService userService;

public void doSomething() {
    User user = userService.getById(1L);  // 像本地方法调用
}

// 实际上是调用了 Dubbo 生成的代理对象：
// proxy.getById(1L) -> 序列化请求 -> 网络发送 -> 等待响应 -> 反序列化结果

```
###### 2.4.3.3 缓存代理

MyBatis 的二级缓存本质上就是一个缓存代理。当执行查询时，CachingExecutor（代理）先查缓存，缓存未命中则委托给真正的 Executor 查库，并将结果放入缓存。也可以自己实现通用的缓存代理：

```java
public class CacheProxy implements DataService {
    private DataService realService;
    private Cache cache;

    public Data query(String key) {
        Data cached = cache.get(key);
        if (cached != null) return cached;
        Data result = realService.query(key);
        cache.put(key, result);
        return result;
    }
}

```
###### 2.4.3.4 保护代理/智能引用
除此之外，还可以保护代理：创建保护代理，实现与目标相同的接口，在方法内先执行权限检查，通过后才委托给真实对象。

智能引用代理：可在没有客户端使用某个重量级对象时立即销毁该对象。

#### 2.5 享元模式

享元模式只有一个目的： 将内存消耗最小化。 如果你的程序没有遇到内存容量不足的问题， 则可以暂时忽略该模式。

实现：
- 享元类中包含原始对象中部分能在多个对象中共享的状态（一般是大内存）。 
- 情景类包含原始对象中各不相同的外在状态。 






### 3. 行为模式
这类模式负责对象间的高效沟通和职责委派。
#### 3.1 责任链模式
允许你将请求沿着处理者链进行发送。 收到请求后， 每个处理者均可对请求进行处理， 或将其传递给链上的下个处理者。
最常用于：**过滤器链**。
- 有一个处理者接口 (Filter)；
- 有一系列具体处理者；
- 有一个链对象负责串联并迭代处理者；
- 请求沿着链传递，每个处理者自主决定是否传递以及如何传递。

#### 3.2 状态模式

让你能在一个对象的内部状态变化时改变其行为， 使其看上去就像改变了自身所属的类一样。
如果不用状态模式，可能会有非常多的if-else，随着状态增加或转换，维护起来不容易。
基本思路：**每种状态作为一个独立类，操作委托给当前状态对象**。
##### 3.2.1 模式架构

![alt](http://image.huawei.com/tiny-lts/v1/images/hi3ms/33b1eac19c8aafb2857567011be1805b_810x615.png)


状态 （State） 接口会声明特定于状态的方法。
**上下文 （Context）** 保存了对于一个具体状态对象的引用， 并会将所有与该状态相关的工作委派给它。 上下文通过状态接口与状态对象交互， 且会提供一个设置器用于传递新的状态对象。

##### 3.2.2 最佳实践
这里以最常见的电商订单状态为例。

```
待支付 → 已支付 → 已发货 → 已收货 → 已完成
    ↓         ↓
  已取消    申请退款 → 退款中 → 已退款

```
状态接口和具体状态类定义如下：

```java
public interface OrderState {
    void pay(OrderContext context);
    void cancel(OrderContext context);
    void ship(OrderContext context);
    void confirmReceive(OrderContext context);
}

public class PendingPaymentState implements OrderState {
    @Override
    public void pay(OrderContext context) {
        // 执行支付逻辑...
        System.out.println("支付成功，订单状态变为已支付");
        context.setState(new PaidState()); // 切换到已支付
    }

    @Override
    public void cancel(OrderContext context) {
        System.out.println("订单已取消");
        context.setState(new CancelledState());
    }

    @Override
    public void ship(OrderContext context) {
        throw new UnsupportedOperationException("待支付订单不能发货");
    }

    @Override
    public void confirmReceive(OrderContext context) {
        throw new UnsupportedOperationException("待支付订单不能确认收货");
    }
}

```
上下文实现：

```java
public class OrderContext {
    private OrderState currentState;

    public OrderContext() {
        this.currentState = new PendingPaymentState(); // 初始状态
    }

    public void setState(OrderState state) {
        this.currentState = state;
    }

    // 委托给当前状态对象
    public void pay() { currentState.pay(this); }
    public void cancel() { currentState.cancel(this); }
    public void ship() { currentState.ship(this); }
    public void confirmReceive() { currentState.confirmReceive(this); }
}

```
客户端调用：

```java
OrderContext order = new OrderContext();
order.pay();              // 支付成功，状态→已支付
order.ship();             // 发货成功，状态→已发货
order.confirmReceive();   // 确认收货，状态→已完成
order.cancel();           // 抛出异常：已完成订单不能取消

```
##### 3.2.3 应用场景

- **如果对象需要根据自身当前状态进行不同行为， 同时状态的数量非常多且与状态相关的代码会频繁变更的话， 可使用状态模式**。
- 如果某个类需要根据成员变量的当前值改变自身行为， 从而需要使用大量的条件语句时， 可使用该模式。

#### 3.3 模板方法模式
模板方法模式的核心是：在一个方法中定义好算法的骨架（固定流程），将其中某些步骤延迟到子类实现。 子类可以重写这些步骤，但不能改变整体结构。

这是一个体现**封装、继承**的模式，就是把大家都共用的方法封装到超类中，不同子类实现的不同方法设置为抽象方法，让子类重写。

##### 3.3.1 最佳实践
###### 3.3.1.1 JUnit 测试框架
JUnit 测试框架是模板方法模式的教科书级案例。
JUnit 3.x 中，TestCase 类为每个测试方法的执行定义了固定流程：
`setUp() → runTest() → tearDown()`；
其中，setUp() 和 tearDown() 是**钩子方法**，默认空实现，留给子类覆盖来做初始化和资源释放；而 runTest() 是**抽象方法**，由具体的测试方法提供。

```java
public class UserServiceTest extends TestCase {
    @Override
    protected void setUp() {
        // 初始化数据库连接、测试数据
    }

    public void testAddUser() {
        // 具体测试逻辑
    }

    @Override
    protected void tearDown() {
        // 清理资源
    }
}

```
###### 3.3.1.2 Spring 中的各种 Template

数据库操作的通用流程是：获取连接 → 执行 SQL → 处理结果 → 释放连接（异常处理）。获取连接、异常处理、资源释放这些是固定不变的，真正变化的是“执行什么 SQL”和“如何处理结果”。

JdbcTemplate 将这些固定部分封装好，将变化部分抽象为回调接口（如 RowMapper、PreparedStatementCallback），使用者只需传入具体的实现：

```java
jdbcTemplate.query("SELECT * FROM users", 
    (rs, rowNum) -> new User(rs.getString("name")));

```
##### 3.3.2 应用场景
当你只希望客户端扩展某个特定算法步骤， 而不是整个算法或其结构时， 可使用模板方法模式。
你希望为框架用户提供简化开发的钩子方法，只让他们关注业务点。

#### 3.4 中介者模式
最常用的思想之一，遇事不决加中间层。
中介者模式用一个中介对象来封装一系列对象之间的交互，避免这些对象显式地相互引用，从而降低耦合度，提升可维护性。关注的是如何实现**多个对象之间如何协调交互而不形成网状依赖**。

##### 3.4.1 最佳实践
**最典型的实现**：聊天室服务器 —— 最经典的网络中介者。

用户（User）之间不直接通信，而是由聊天室（ChatRoom）服务器充当中介者，负责消息转发、用户上线/下线管理等。


```java
// 中介者接口
interface ChatRoom {
    void showMessage(User user, String message);
    void addUser(User user);
}

// 具体中介者
class ChatRoomImpl implements ChatRoom {
    private Map<String, User> users = new HashMap<>();

    @Override
    public void addUser(User user) {
        users.put(user.getName(), user);
        user.setRoom(this);
    }

    @Override
    public void showMessage(User user, String message) {
        String formatted = "[" + user.getName() + "]: " + message;
        // 向所有在线用户广播（此处可以结合观察者模式）
        for (User u : users.values()) {
            if (!u.equals(user)) { // 通常不给自己回显
                u.receive(formatted);
            }
        }
    }
}

// 同事类：用户
class User {
    private String name;
    private ChatRoom room;

    public User(String name) { this.name = name; }
    public String getName() { return name; }
    public void setRoom(ChatRoom room) { this.room = room; }

    public void send(String message) {
        room.showMessage(this, message); // 通过中介者发送
    }

    public void receive(String message) {
        System.out.println(name + " 收到: " + message);
    }
}

// 使用
ChatRoomImpl room = new ChatRoomImpl();
User alice = new User("Alice");
User bob = new User("Bob");
room.addUser(alice);
room.addUser(bob);

alice.send("Hello Bob!");
bob.send("Hi Alice!");

```
中介者常常可配合观察者模式实现，聊天室服务器既充当中介者协调用户交互，又使用观察者模式向在线用户广播消息。

#### 3.5 迭代器模式
这个不多说，很多语言底层都用到了这个模式。
迭代器模式让你能在不暴露集合底层表现形式 （列表、 栈和树等） 的情况下遍历集合中所有的元素。

#### 3.6 命令模式
##### 3.6.1 问题和意图
命令模式可将请求转换为一个包含与请求相关的所有信息的独立对象。 该转换让你能根据不同的请求将方法参数化、 延迟请求执行或将其放入队列中， 且能实现可撤销操作。
简单说，将命令封装成一系列命令类，发送者和接收者之间通过命令交互。
##### 3.6.2 模式结构

![alt](http://image.huawei.com/tiny-lts/v1/images/hi3ms/ceeeb168666fc9777ebdebc8dad50c2c_945x555.png)


客户端 （Client） 会创建并配置具体命令对象。 客户端必须将包括接收者实体在内的所有请求参数传递给命令的构造函数。 此后， 生成的命令就可以与一个或多个发送者相关联了。

发送者 （Sender）——亦称 “触发者 （Invoker）”——类负责对请求进行初始化， 其中**必须包含一个成员变量来存储对于命令对象的引用**。 发送者触发命令， 而不向接收者直接发送请求。 注意， 发送者并不负责创建命令对象： 它通常会通过构造函数从客户端处获得预先生成的命令。

具体命令 （Concrete Commands） 会实现各种类型的请求。 **具体命令自身并不完成工作， 而是会将调用委派给一个业务逻辑对象**。 但为了简化代码， 这些类可以进行合并。

##### 3.6.3 代码实现
命令模式最常实现的操作之一，就是撤销与重做操作。

```java
// 命令接口
public interface Command {
    void execute();
    void undo();
}

// 接收者：文本编辑器
public class TextEditor {
    private StringBuilder text = new StringBuilder();
    public void insert(int pos, String str) { text.insert(pos, str); }
    public void delete(int pos, int len) { text.delete(pos, pos + len); }
    // ... 获取状态用于undo
}

// 具体命令：插入文字
public class InsertCommand implements Command {
    private TextEditor editor;
    private int pos;
    private String str;
    
    public InsertCommand(TextEditor editor, int pos, String str) { ... }
    
    public void execute() {
        editor.insert(pos, str);
    }
    public void undo() {
        editor.delete(pos, str.length());
    }
}

// 具体命令：删除文字（保留删除内容以便undo）
public class DeleteCommand implements Command {
    private TextEditor editor;
    private int pos;
    private String deleted; // 快照
    public void execute() {
        deleted = editor.getText().substring(pos, length);
        editor.delete(pos, length);
    }
    public void undo() {
        editor.insert(pos, deleted);
    }
}

// 调用者：命令历史管理器
public class CommandHistory {
    private Stack<Command> undoStack = new Stack<>();
    private Stack<Command> redoStack = new Stack<>();
    
    public void execute(Command cmd) {
        cmd.execute();
        undoStack.push(cmd);
        redoStack.clear(); // 新操作清空重做栈
    }
    public void undo() {
        if (!undoStack.isEmpty()) {
            Command cmd = undoStack.pop();
            cmd.undo();
            redoStack.push(cmd);
        }
    }
    public void redo() {
        if (!redoStack.isEmpty()) {
            Command cmd = redoStack.pop();
            cmd.execute();
            undoStack.push(cmd);
        }
    }
}

// 客户端：菜单和快捷键都只触发命令
Command insertCmd = new InsertCommand(editor, 0, "Hello");
history.execute(insertCmd);  // 执行
history.undo();             // 撤销
history.redo();             // 重做

```
##### 3.6.4 最佳实践：异步任务调度
**命令模式是异步任务调度的基石**。
多线程编程中，需要将任务提交到线程池异步执行。任务由不同业务逻辑产生，但线程池只需通用的调度执行机制。

在没有线程池的原始并发编程中，通常直接创建线程并传入任务，没有命令模式封装：发送者直接依赖接收者和执行细节。

命令模式将任务抽象为可传递的对象，线程池则提供了强大的调度器，两者结合才使得高并发编程从混乱走向秩序.

Java 的 Runnable 和 Callable 接口就是命令接口，具体业务任务就是具体命令。线程池扮演调用者/调度者角色，工作线程是接收者。

###### 3.6.4.1 命令接口

```java
// 命令接口 - 只定义一个 execute() 方法，此处名为 run()
@FunctionalInterface
public interface Runnable {
    void run();  // 对应 Command 模式的 execute()
}

// 如果需要返回值和抛出异常，使用更强大的命令接口
public interface Callable<V> {
    V call() throws Exception;
}

```
###### 3.6.4.2 具体命令

```java
// 具体命令：发送邮件任务，内部持有接收者 EmailService
public class SendEmailTask implements Runnable {
    private final EmailService emailService;  // 接收者
    private final String to;
    private final String content;

    public SendEmailTask(EmailService emailService, String to, String content) {
        this.emailService = emailService;
        this.to = to;
        this.content = content;
    }

    @Override
    public void run() {
        // 委托给接收者执行真正的业务逻辑
        emailService.send(to, content);
    }
}

```
在现代 Java 中，我们更常用 lambda 表达式来简化这个具体命令的定义，但本质未变——lambda 仍然是一个实现了 Runnable 接口的匿名命令对象：

```java
Runnable task = () -> emailService.send(to, content);

```
###### 3.6.4.3 调用者：线程池

ThreadPoolExecutor 是命令模式中最经典的调用者实现。它内部维护了一个命令队列（BlockingQueue<Runnable>）和一组工作线程（Worker）。

工作线程（Worker）在 runWorker 中无限循环，反复调用 getTask() 从队列中取出命令。取出命令后，工作线程调用 task.run()，执行命令，从而触发接收者的对应操作。
##### 3.6.5 应用场景
当你需要实现 **可撤销操作、任务队列、宏/批处理、命令日志与重放** 等功能时，命令模式是最核心的抽象手段。

#### 3.7 策略模式
##### 3.7.1 问题与意图
策略模式的核心是定义一系列算法，将每个算法封装起来并让它们可以互相替换，使得算法的变化独立于使用它的客户端。它主要用来解决“同一行为有多种不同实现，且需要动态选择”的问题，避免大量 if-else 分支。
##### 3.7.2 模式结构
![alt](http://image.huawei.com/tiny-lts/v1/images/hi3ms/51e2aa9d26eae70576d201b21834c7e8_660x555.png)


客户端 （Client） 会创建一个特定策略对象并将其传递给上下文。 上下文则会提供一个设置器以便客户端在运行时替换相关联的策略。
一般来说，客户端必须知晓策略间的不同——它需要选择合适的策略。

##### 3.7.3 代码实现
这里用数据压缩与加密 —— 算法族替换来举例实现。
策略模式最大的好处是让算法独立于客户端而变化，它特别适用于“同一行为，多种算法”的场景，让系统更易扩展、更易测试。

```java
// 压缩策略接口
public interface CompressionStrategy {
    byte[] compress(byte[] data);
    byte[] decompress(byte[] data);
}

// GZIP 压缩策略
public class GzipCompressionStrategy implements CompressionStrategy {
    public byte[] compress(byte[] data) { /* GZIP 实现 */ }
    public byte[] decompress(byte[] data) { /* GZIP 解压 */ }
}

// ZIP 压缩策略
public class ZipCompressionStrategy implements CompressionStrategy { ... }

// 上下文：文件处理器
public class FileProcessor {
    private CompressionStrategy compressionStrategy;
    public void setCompressionStrategy(CompressionStrategy strategy) {
        this.compressionStrategy = strategy;
    }
    public void save(String path, byte[] data) {
        byte[] compressed = compressionStrategy.compress(data);
        // 写入文件
    }
}

```
##### 3.7.4 最佳实践
工厂模式和策略模式经常搭配使用，而且是非常经典的组合。

策略模式负责“如何做”，工厂模式负责“选哪一个”。将策略作为工厂的产品，可以在不暴露具体策略类选择逻辑的情况下，动态获取策略实例，彻底消除客户端里的 if-else 或 switch。
这里以支付为例：

```java
// 策略工厂
public class PaymentStrategyFactory {
    private Map<String, PaymentStrategy> strategyMap = new HashMap<>();

    public PaymentStrategyFactory() {
        // 初始化所有策略，这里可以用反射或SPI进一步增强
        strategyMap.put("wechat", new WechatPayStrategy());
        strategyMap.put("alipay", new AlipayStrategy());
    }

    public PaymentStrategy getStrategy(String channel) {
        PaymentStrategy strategy = strategyMap.get(channel);
        if (strategy == null) {
            throw new IllegalArgumentException("Unsupported channel: " + channel);
        }
        return strategy;
    }
}

// 支付策略接口
public interface PaymentStrategy {
    PayResult pay(PayRequest request);
}

// 微信支付策略（适配微信SDK）
public class WechatPayStrategy implements PaymentStrategy {
    public PayResult pay(PayRequest request) {
        // 1. 转换 PayRequest -> 微信统一下单请求
        // 2. 调用微信 SDK 发起支付
        // 3. 转换微信响应 -> PayResult
    }
}

// 支付宝支付策略
public class AlipayStrategy implements PaymentStrategy {
    public PayResult pay(PayRequest request) {
        // 类似委托给支付宝 SDK
    }
}

// 上下文
public class PaymentService {
    public PayResult executePayment(PayRequest request, PaymentStrategy strategy) {
        // 可以加入日志、校验等公共逻辑
        return strategy.pay(request);
    }
}

// 客户端使用
PaymentStrategy strategy = factory.getStrategy("wechat");
paymentService.executePayment(request, strategy);

```

在支持高阶函数的语言中，**策略模式可以缩减为直接传递函数**,“具体策略类”被内联成了 **lambda 表达式**，减少了样板代码，但思想完全一致。如：

```java
// Java 8+ 用 Function 接口替代策略接口
import java.util.function.Function;

public class CheckoutService {
    public BigDecimal settle(Order order, Function<Order, BigDecimal> discountFn) {
        BigDecimal discount = discountFn.apply(order);
        return order.getTotalAmount().subtract(discount);
    }
}

// 使用时直接传入 lambda
checkout.settle(order, o -> o.getTotalAmount().multiply(new BigDecimal("0.2")));  // 折扣策略
checkout.settle(order, o -> new BigDecimal(50));  // 固定立减策略

```
##### 3.7.5 应用场景
当你想使用对象中各种不同的算法变体， 并希望能在运行时切换算法时， 可使用策略模式。

#### 3.8 观察者模式
##### 3.8.1 问题和意图

观察者模式是一种行为设计模式， 允许你定义一种订阅机制， 可在对象事件发生时通知多个 “观察” 该对象的其他对象。
##### 3.8.2 模式结构

![alt](http://image.huawei.com/tiny-lts/v1/images/hi3ms/7ea91ebf01633f1e3cf5982274822389_915x465.png)


发布者和订阅者通过事件驱动。当新事件发生时， 发送者会遍历订阅列表并调用每个订阅者对象的通知方法。 该方法是在订阅者接口中声明的。
##### 3.8.2 最佳实践
观察者模式早已不限于一对多的回调接口，更体现为**事件驱动架构的基石**。
观察者模式最重要的是它的思想而不是实现，它提示我们**用事件通知替代主动调用**，实现解耦。
事件驱动架构 将这一思想扩展到系统层面，用事件通道和异步机制构建松耦合的分布式系统。
例如：
- Netty在网络 I/O 框架内部，完美展示了如何用事件驱动模型高效处理并发连接和数据传输。
- Spring 事件机制（ApplicationEvent + @EventListener）
最“内功”的一集。

##### 3.8.3 应用场景
当一个对象状态的改变需要改变其他对象， 或实际对象是事先未知的或动态变化的时， 可使用观察者模式。
