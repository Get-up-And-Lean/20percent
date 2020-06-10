# 主题 01：如何设计一个好的 API

### 1. 引言

如果说好的 UI 设计可以让用户更容易地使用一款产品，那么，好的 API 设计则可以让其他开发者更高效地使用一个系统的能力。良好的 API 可以很大程度上减轻使用者的负担，同时也可以极大地减轻技术支持的工作量，尤其是对那些使用者众多的 API 来说。

在实践中，一个较复杂的系统通常由多位开发者共同开发。往往由于缺乏统一的规范，开发者各自为政，导致同一个系统提供的 API 风格相差甚远，增加使用者的学习成本和后期维护成本。此外，有些时候由于开发资源紧张，可能无法投入足够的资源到 API 的设计、完善和相关文档上，进而导致产出的 API 质量差，难以使用。如是种种，无论对使用者还是维护者都将是一场噩梦。那么，怎样才能设计出良好的 API 呢？

Google 的首席 Java 架构师 Joshua Bloch 在一个演讲中分享了关于 API 设计的理念，他认为一个好的 API 应该具备以下特点：

- Easy to learn and memorize（易学易记）
- Easy to use, even without documentation（易用）
- Hard to misuse（不容易用错）
- Easy to evolve（容易扩展迭代）
- Sufficiently powerful to satisfy requirements（足以满足需求）

### 2. 设计一个好的 API 需要注意的点

本文末尾有 Joshua Bloch 的演讲 PPT 和视频链接。Joshua Bloch 分享的关于 API 的设计理念令人印象深刻，那么，如何在实践中将这些优秀的理念“落地”呢？在我看来有以下需要注意的点。

#### **2.1 明确边界**

在写文章的时候，通常需要首先确定一个主题，然后再围绕主题展开。有了主题的指引，在行文时有利于理清思路：哪些内容与主题相关？哪些内容可以升华主题？既定内容是否跑题？与之类似，设计 API 的时候，我们需要首先明确边界(boundary)，聚焦 API 需要提供的本质能力，避免陷入具体场景而沦为某个业务的专属 API。

![在这里插入图片描述](https://images.gitbook.cn/844ca0a0-07b9-11ea-9755-dbbfca9e4745)

上图是一个简要的系统边界示意图，关于边界，在设计 API 时需要注意以下事项：

1. 只有绿色的部分才是设计 API 所需要考虑的，它是软件系统具体可提供的服务或者能力。API 是系统和外部交互的接口，至于外部如何使用这个接口、通过什么途径使用不应该在我们的考虑范畴。
2. 设计 API 时不应该陷于具体的通信协议。通信协议只是一种信息交换的渠道，而随着技术的发展，这些协议的变动性很大，而 API 的外观相对要稳定得多。
3. 设计 API 不应陷于 UI 交互的相关细节。交互属于客户端的范畴，随着终端设备的多样性，客户端的交互也是趋于多样性和不稳定的。

**举例解读**

在超市结账的时候，当收银员扫描商品的二维码时，POS 终端上就会显示这个产品的价格和名称，那么这个 API 应该如何设计呢？如果一开始选择 REST 架构来做项目，那么很可能会出现上面注意事项 2 所描述的问题——API 和具体的通信协议层代码捆绑。

```java
@Path("/items")
public class ItemResource {
   @RequestMapping("/checkout")
   public ItemCheckoutResult checkoutItem(@RequestParam(value="Barcode") String barcode) {
       // 具体实现代码...
   }
}
```

某种程度上，上面的这种设计在最初并没有什么问题，但随着系统的不断迭代，可能需要支持不同的通信协议，比如 WebSocket、RPC；同时需要支持的终端设备也在增加，比如需要支持手机 App，那么上面的设计会让边界变得越来越模糊，最终可能导致 API 的实现逻辑代码被 copy/paste 得到处都是——repeat yourself everywhere。为了避免上面的情况出现，设计 API 时应明确边界，保证 API 具有良好的独立性，示例代码如下所示：

```java
public interface StoreService {
   ItemCheckoutResult checkoutItem(String barcode);
}
```

然后在协议层的代码中进行调用。

```java
@Path("/items")
public class ItemResource {
   @RequestMapping("/checkout")
   public ItemCheckoutResult checkoutItem(@RequestParam(value="Barcode") String barcode) {
       return storeService.checkoutItem(barcode);
   }
}
```

#### 2.2 Tell, Don't Ask

Tell-don't-ask 原则最早是在 IEEE 软件专栏的[一篇文章](http://media.pragprog.com/articles/jan_03_enbug.pdf)中提出的，某种程度上，它反映了面向过程编程与面向对象编程的本质区别。其核心思想为：在面向对象编程时，应该根据对象的行为来封装具体的业务逻辑，调用方应该直接告诉（tell）对象需要做什么，而不是通过询问（ask）对象的每一个状态然后再告诉对象需要做什么。两种方式的区别如下图所示：

![在这里插入图片描述](https://images.gitbook.cn/6d9f8b80-07bc-11ea-bef3-6307f0c39f3e)

**举例解读**

按照这个原则来设计 API 可以更好地体现软件的系统能力，而避免沦为简单的增、减、改、查操作。为了让读者更好地理解，这里举一个银行取款的例子。

**方案一：按照 ask 模式来设计 API**

step-1，需要创建一个账户对象，如 AskAccountDTO：

```java
public class AskAccountDTO {
    private int id;
    private long balance;
    private long credit;
    private long debt;

    public int getId() {return id;}
    public long getBalance() {return balance;}
    public void setBalance(long balance) {this.balance = balance;}
    public long getCredit() {return credit;}
    public void setCredit(long credit) {this.credit = credit;}
    public long getDebt() {return debt;}
    public void setDebt(long debt) {this.debt = debt;}
}
```

step-2，创建两个 API，分别用于读取和更新这个账户对象。

```java
public interface BankService {
    AskAccountDTO getAccountById(int id);
    void updateAccount(AskAccountDTO account);
}
```

step-3，调用方来实现取钱逻辑。

```java
    // 用户账户 ID 和取款数
    int id = 20881234;
    long amount = 500;

    AskAccountDTO account = bankService.getAccountById(id);
    if (account.getBalance() >= amount) {
        account.setBalance(account.getBalance() - amount);
        bankService.updateAccount(account);
        return;
    }
    long total = account.getBalance() + account.getCredit();
    if (total >= amount) {
        long restAmount = amount - account.getBalance();
        account.setBalance(0);
        account.setDebt(account.getDebt() + restAmount);
        bankService.updateAccount(account);
        return;
    }
    throw new InsufficientBalanceException("您的账户资金不足");
```

**方案二：按照 tell 模式来设计 API**

step-1，只需要一个API——“withdraw”，在这个 API 内部实现所有的取款逻辑（同方案一 step-3 中调用方实现的代码逻辑）。

```java
public interface BankService {
    TellAccountDTO withdraw(int id, long amount);
}
```

step-2，方案一中的账户对象简化后作为这个 API 的出参，用以承载取钱操作的返回信息。

```java
public class TellAccountDTO {
    private int id;
    private long balance;
    private long credit;
    private long debt;

    public int getId() {return id;}
    public long getBalance() {return balance;}
    public long getCredit() {return credit;}
    public long getDebt() {return debt;}
}
```

step-3，调用方只需要 “tell” 这个 API 需要取款，API 内部完成所有的计算和判断。

```java
TellAccountDTO account = bankService.withdraw(20881234, 500);
```

#### **2.3 Do One Thing**

“Do One Thing”—— 即单一职责。在设计 API 的时候，力求一个 API 只做一件事情，单一职责不但可以让 API 的外观更稳定、没有歧义（side effects）、简单易用，而且可以提高 API 的可重用性。在设计 API 的时候，如果符合以下条件，可以考虑拆分：

1. 一个 API 可以完成多个功能。例如，一个 API 既可以用于修改商品的价格，又可以修改商品标题、描述、库存等，通常这些功能并不需要在一次调用里完成，修改价格的时候通常不会去修改标题和描述，合并在一起会使得接口过于复杂，不易使用。
2. 一个 API 用于处理不同类型的对象。例如，发布不同类型的商品可以拆成多个 API，这样可以简化数据结构，发布服装类商品为什么要关心卡券类商品的属性（如有效期）呢。

**举例解读**

通过用户名和密码进行登录是一个很常见的功能，一般通过设计一个 login 方法来实现，示例代码如下。

接口示例：

```java
public interface SomeService {
  String login((String username, String password);
}
```

实现示例：

```java
public class SomeServiceImpl implements SomeService{
  @Override
   public String login(String username, String password) {
        User user = userRepository.findByUsername(username);
        if (null == user) {
            // 略
        }
        if (!user.verifyPassword(password)) {
            // 略
        }
        Session session = sessionFactory.generate(user);
        return session.getKey();
    }
}
```

看上去这个 API 没有什么问题，而且，也满足“tell-dont-ask”原则。但是这个方法内部其实做了两件事情：

1. 检验用户名和密码的正确性，并且返回相应结果；
2. 如果用户名和密码验证成功，则创建一个用户 session。按照“do one thing”的原则来设计这个功能，应该把这两件事情变成两个 API，示例代码如下。

接口示例：

```java
public interface SomeService {
  boolean verifyUserCredential(String username, String password);
  String createUserSession(String username);
}
```

实现示例：

```java
public class SomeServiceImpl implements SomeService{
    @Override
    public boolean verifyUserCredential(String username, String password) {
        User user = userRepository.findByUsername(username);
        if (null == user) {
            return false;
        }
        if (!user.verifyPassword(password)) {
            return false;
        }
        return true;
    }
    @Override
    public String createUserSession(String username) {
        User user = userRepository.findByUsername(username);
        if (null == user) {
            //抛出用户未找到异常
        }
        Session session = sessionFactory.generate(user);
        return session.getKey();
    }
}
```

使用示例：

```java
if (someService.verifyUserCredential("zhangSan", "2088124567")) {
    String sessionKey = someService.createUserSession("zhangSan");
}
```

上述设计的好处是 verifyCredential 和 createUserSession 可以被分别独立使用，在某些场景下也许我们只需要为用户创建一个新的 session 而不一定需要再次输入用户名和密码，反之亦然。

#### **2.4 不要基于实现设计 API**

在设计 API 的时候，要避免陷入实现细节，API 应该与实现无关，它不能泄露实现相关的信息，以免误导用户。什么是实现细节呢？如过多地透露 API 的行为，以常见的 hash 方法为例，其实现方式很多（直接定址法、除留余数法、平方取中法、折叠法等），设计 API 时不应透露 hash 方法的实现方式。

**bad：**

```java
public interface HashService {
    int hashBasedOnDirectAddr(Object key);
}
```

**good：**

```java
public interface HashService {
    int hash(Object key);
}
```

#### **2.5 Exception Or Error Code?**

系统运行过程中难免出现异常，那么就 API 的设计而言，是抛出异常还是返回错误码呢？关于这个问题，业内争议不断，在我看来，两种方式并没有绝对的高下之分。不论是 exception 还是 error code，核心点在于当 API 产生错误的时候，API 的调用方是否可以清晰地理解错误信息，并据此做出正确的处理。

在复杂的系统中，error code 有一定优势。API 调用具有复杂的多层级调用关系——一个系统的调用者还会被其他系统调用，要一层层的抛出错误。如果采用 exception，调用层次太多时将难以分类，如果下游系统不能分类，上游也将无法为调用者分类，到终端调用者时，已经不知道该如何处理这个错误了，这种情况通常只能找维护人员解决。

当然，复杂系统采用 error code 的前提是错误处理需要有统一的规范，以下是几种常见的形式：

1. {"message": "xxx", "code": "200", "success": true}
2. {"message": "xxx", "code": "XXX_EXCEPTION_ERROR", "success": false}
3. {"code": 500, "error": "msg xxx"}

**使用 Exception**

如下例子，API 的设计中使用了 unchecked exception。

接口示例：

```java
public interface SomeService {
    String createUserSession(String username) ;
}
```

实现示例：

```java
public class SomeServiceImpl implements SomeService{
    @Override
    public String createUserSession(String username) {
        User user = userRepository.findByUsername(username);
        if (null == user) {
            throw new BusinessLogicException(40018, "no user found with given username");
        }
        Session session = sessionFactory.generate(user);
        return session.getKey();
    }
}
```

异常 BusinessLogicException 定义。

```java
public class BusinessLogicException extends RuntimeException {
    private int errorCode;
    public BusinessLogicException(int errorCode, String msg) {
        super(msg);
        this.errorCode = errorCode;
    }
    public int getErrorCode() {return errorCode;}
}
```

使用示例：

```java
String sessionKey = someService.createUserSession("testUser");
```

**使用 Error Code**

接口定义：

```java
public interface SomeService {
    SessionResult createUserSession(String username) ;
}
```

实现示例：

```java
public class SomeServiceImpl implements SomeService {
    @Override
    public SessionResult createUserSession(String username) {
        SessionResult result = new SessionResult();
        User user = userRepository.findByUsername(username);
        if (null == user) {
            result.setSuccess(false);
            result.setErrorCode("NO_USER_FOUND");
            result.setErrorDesc("no user found with given username");
            return result;
        }
        Session session = sessionFactory.generate(user);
        result.setSessionKey(session.getKey());
        return result;
    }
}
```

SessionResult 定义：

```java
public class SessionResult extends CommonResult {
    private String sessionKey;
    public String getSessionKey() {
        return sessionKey;
    }
    public void setSessionKey(String sessionKey) {
        this.sessionKey = sessionKey;
    }
```

CommonResult 的定义：

```java
public class CommonResult implements Serializable{
    // 序列化相关省略
    private boolean  success = true;
    private String  errorCode;
    private String  errorDesc;
    // getter、setter 省略
```

#### **2.6 避免 Flag 效果的参数**

在设计 API 时，为了兼容不同的逻辑分支，有时会通过增加一个参数来实现不同分支的切换。如下示例：读取学生信息的 API 设计。

```java
public interface SchoolService {
    PaginatedResult<List<StudentDTO>> listStudents(boolean isGraduated);
}
```

上面的设计并没有太大的问题，只是对 API 的调用方并不十分友好，在使用 API 的时候，参数 isGraduated 的作用可能会让调用方疑惑。其实，我们完全可以将上面的 API 设计成如下形式，清晰明了：

```java
public interface SchoolService {
    PaginatedResult<List<StudentDTO>> listInSchoolStudents();
    PaginatedResult<List<StudentDTO>> listGraduatedStudents();
}
```

#### **2.7 名字很重要，API 即语言**

在设计 API 的时候，还应给它取一个合适的名字，这样调用方在使用的时候会更容易。关于 API 命名，通常需要注意以下几个方面：

1. API 的名字应该能自解释，即 API 的名字本身就可以很好地描述 API 的能力；
2. 保持一致性，如 callback 应该在同系统所有的 API 中表示一样的意思；
3. 保持对称性，如 set/get、read/write；
4. 拼写准确，API 发布之后便无法更改，只能增加新的 API，直到旧 API 无人使用了才能废弃，因此发布的时候要注意检查拼写。如将 capacity 错写成 capicity。

**good：**

```java
public interface SchoolService {
    boolean addStudentToCourse(long studentId, String courseCode);
}
```

**better：**

```java
public interface SchoolService {
    boolean enrollCourse(long studentId, String courseCode);
}
```

#### **2.8 使用尽量精确的数据结构**

在设计 API 的时候，应尽量使用精确的数据结构，避免为复用而复用（其实是为偷懒），复用的一些数据结构可能与 API 本身并不十分匹配，甚至存在一些对使用者毫无意义的字段，导致使用者难以理解。

举个例子：

> 编辑单个商品 SKU 的库存和上线商品共用一个返回结果模型，但前者是单体操作，后者是批量操作，为了兼容这两种操作，返回对象里既包含单个的商品 id，又包含商品 id 列表；与此同时，错误信息里既包含单个错误信息，又包含一个错误信息列表。如此设计，无形中增加了调用方的学习成本，降低了效率。

#### **2.9 给 API 建立文档**

好的 API 也需要好的文档，否则也有可能“收获”骂声一片。同时，要站在用户的角度去写文档，而不是开发者的视角——“这个接口很简单，说明略”。API 的文档如同一份”合约“，不只是让 API 的使用方更容易使用和理解，更重要的是，让 API 的提供方按照这份”合约“来保证 API 的实现是对的。通常 API 的文档应包含以下内容：

1. Maven 依赖
2. 类、方法、参数、异常、错误码详细说明
3. 使用范例（针对不同场景分别举例）
4. 历史版本
5. FAQ
6. 注意及时更新

#### **2.10 统一的规范**

同一个系统提供的不同 API 之间应该遵循统一的规范，保持一致的风格，这样不仅有助于降低使用者的学习成本，而且可为后续迭代开发提供可遵循的范式。

在实践中，同一个系统往往由众多的开发者共同开发，如果没有统一的规范，开发者都按照自己的习惯设计、开发，那么，这样的系统无论对使用者还是维护者都将是一场噩梦。举个反例，数年前，在笔者参与开发的一个系统中，由于事先没有约定规范（统一的 API 模板），不同开发者对 API 的返回结果中错误码字段的定义完全不同，有的 API 用 errorCode，有的用 resultCode，有的用 code，有的甚至没有错误码字段。系统交付后，“收获”抱怨声一片。

### 3. 总结

关于如何设计一个好的 API，业界大牛们提出了很多优秀的设计理念，但是在实践中将这些优秀的理念“落地”却是相对困难的。就本文提及的“do one thing”和“tell-don't-ask”原则，两者之间就存在矛盾，对于经验不够丰富的工程师，如何在二者之间取得平衡是一个难题。

事实上，“do one thing”和“tell-don't-ask” 的侧重点是不一样的。“tell-don't-ask” 的关注点在服务层（按照 DDD 的说法就是应用层）的接口设计粒度应该做到“tell”，如上文中银行取钱（widthdraw 方法）的示例。而“do one thing”的侧重则在于保持代码的可维护性、可重用性、可测试性，在“widthdraw 方法”内部实现的时候，再按照“do one thing”原则把代码划分为独立的 method（getAccountById 和 updateAccount）进行组织。

不同的设计原则侧重会有不同，但并不是绝对的隔离，而是相辅相成的。

### 4. 参考文献

- [How to Design a Good API and Why it Matters](http://www.cs.bc.edu/~muller/teaching/cs102/s06/lib/pdf/api-design?spm=ata.13261165.0.0.38067cd1w5lGp5)
- 视频：[How To Design A Good API and Why it Matters](http://v.youku.com/v_show/id_XMzA0NjY0MjAw.html?spm=ata.13261165.0.0.38067cd1w5lGp5)