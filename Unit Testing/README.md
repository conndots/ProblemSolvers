如何正确地单元测试？
===============
# 什么是单元测试？
在Wikipedia的[相关页面](https://zh.wikipedia.org/wiki/%E5%8D%95%E5%85%83%E6%B5%8B%E8%AF%95)中有如下的定义：  
>在计算机编程中，单元测试（又称为模块测试, Unit Testing）是针对程序模块（软件设计的最小单位）来进行正确性检验的测试工作。程序单元是应用的最小可测试部件。  
  
单元测试的目标是隔离程序部件并证明这些单个部件是正确的。编写单元测试是一种**验证行为**，更是一种**设计行为**。它更是以各种**编写文档的行为**。编写单元测试避免了相当数量的反馈循环，尤其是功能验证方面的。
  
单元测试已经成为了极限编程（eXtreme Programming）的标识性行为，并很快发展成为了**测试驱动开发方法**。  
  
同样的，也有**单测覆盖率**的概念。基本的覆盖率准则如下：  
  
1. 函数覆盖率（Function coverage）：有没有覆盖到程序中的每一个函数；  
2. 指令覆盖率（Statement coverage）：若用控制流图表示程序，单测有执行到控制流图的每一个节点吗；  
3. 判断覆盖率（Decision coverage）：若用控制流图表示程序，有执行到途中的每一个边吗？如if指令都有执行到逻辑运算式成立与不成立的情形吗？  
4. 条件覆盖率（Condition voverage）：每一个逻辑运算式中的每一个条件（无法再分解的逻辑运算式）是否都有执行到成立及不成立的情形吗？条件覆盖率成立不表示判断覆盖率一定成立；  
5. 条件/判断覆盖率（Condition/decision coverage）：需同时满足判断覆盖率和条件覆盖率。  
  
如下的Java代码：  
待测试的类的方法：  
  
```java
public class IDCheckingService {
    @Autowired
    private IdService idService;
    
    public boolean isValidIDNumber(String idNumber) {
        return idNumber != null && idNumber.length == 18 && (idNumber
                .matches("[0-9]{18}") || idNumber.matches("[0-9]{17}[a-zA-Z]"); 
    }
    
    public boolean isNameAndIDMatch(String name, String idNumber) {
        if (idNumber == null || name == null) {
            return false;
        }
        String name2ID = idService.getNameByIDNumber(idNumber);
        return name2ID != null && name2ID.equals(name);
    }
    
    public boolean isValidPerson(String name, String idNumber) {
        if (name == null || name.length() == 0 || name.length() > 4) {
            return false;
        }
        return isValidIDNumber(idNumber) && isNameAndIDMatch(name, idNumber);
    }
    
    public void setIdService(IdService idService) {
        this.idService = idService;
    } 
}
```  
  
它的测试类：  
  
```java
public class IDCheckingServiceTest {
    @Test
    public void isValidPersonTest() {
        IdService idService = IdService.getInstance();
        IdCheckingService idCheckingService = new IdCheckingService();
        idCheckingService.setIdService(idService);
        
        assertTrue(isValidPerson("谢小萌", "23417819931221232X"));
        assertTrue(isValidPerson("郝大龙", "34423319930918343x"));
        assertTrue(isValidPerson("李大锤", "14010619930613762x"));
        assertFalse(isValidPerson(null, "14312419920707123x"));
    }
}
```  
  
对于上面的类和测试代码，函数覆盖率只有1/3，指令覆盖率有100%， 而判断覆盖率是100%，但条件覆盖率并不是100%，许多的逻辑表达式中的子表达式并没有被计算。  
  
另外，上面的类中包含了外部依赖IdService的实例，很可能是需要查询数据库的服务。为了使得IdCHeckingService的单测能够运行，初始化了一个新的IdService，并新加了一个setIdService的方法。新加的方法暴露了内部实现，并不是一种好的做法。这种针对包含外部依赖的单元测试的做法也值得商榷。  
  
刻意追求高的单测覆盖率甚至作为团队的指标，也是一种错误的追求，高的单元测试覆盖率应该是每个“认真”写单元测试的程序员得到的必然结果。在InfoQ的这篇文章[*测试覆盖率有什么用？*](http://www.infoq.com/cn/articles/test-coverage-rate-role)中提到一种说法：   
  
>测试覆盖是一种“学习手段”。学习什么呢？学习为什么有些代码没有被覆盖到，以及为什么有些代码变了测试却没有失败。理解“为什么”背后的原因，程序员就可以做相应的改善和提高，相比凭空想象单元测试的有效性和代码的好坏，这会更加有效。  
  
# 为什么要单元测试？  
极限编程倡导的程序开发方法是测试驱动开发（Test Driven Development, TDD）。意思即是，先写要实现方法的测试方法，然后再编码实现这个功能。前面提到过，单元测试更是一种设计行为，与编写文档的行为。这样的做法便显得非常合理：首先设计好方法的功能的所有细节，然后再去实现它。并且这样的“文档”是自动化验证的，每次代码有变化、更新后，都能通过自动化的单元测试结果发现其功能与初始设计的意图有没有违背，保证不会导致bug。程序的每一项功能都有测试来验证它们操作的正确性。TDD所倡导的理念是：**实现代码是刚好够通过测试的最简单代码**。  
  
另外，在软件开发与进化过程中，除了初期的设计行为，重构行为也是一种提高代码质量、提高代码可读性、扩展性、可维护性与可测性的重要手段。在大神Martin Fowler的书籍《重构》中强调了，每次重构，必须保证重构的方法有充足的单元测试，重构后与重构前的行为表现要一致，保证重构的质量。  
  
除此之外，测试驱动的开发方法还有许多好处：  
  
* 编写单元测试可以迫使程序作者使用不同的观察点重新审视自己要实现的功能，即从调用者与使用者的角度。这样，编写者除了关注实现的功能与方法，也会关注方法的接口。  
* 这种开发方法使得程序变为可测试的。为了成为易于调用与可测试的，程序必须与它的周边环境解耦。测试驱动开发方法有利于开发者解除软件中的耦合。先编写测试代码，常常可以暴露程序中应该被解耦合的区域。上面的代码例子便有外部依赖，有耦合。  
* 另外，方法的其他调用者也可以通过观察方法对应的单元测试代码来学习如何使用方法的接口。  
  
通过测试也能帮助我们发现更好的实现方式，帮助解耦合的机会，提供了重构的可能；测试也能保证重构的正确性。  
  
# 单元测试是正确的，如何正确地单元测试？
## 传统的单元测试        
在传统的单元测试中，我们以上面的几种覆盖率为目标，提高测试的范围，编写测试代码。这些方法的确可以帮我们找到一些显而易见的代码冗余或者测试遗漏的问题。  
  
但仍然有许多经验之谈是需要注意的。  
### 单元测试中的外部依赖与框架依赖  
一般的测试方法可以分为三类：  
  
1. 对无外部依赖方法的测试。  
2. 对有外部依赖方法的测试。    
3. 对依赖于框架的组件的测试。

对于第二种情况，包含对其他组件、第三方库、对文件系统、数据库等有通讯请求的方法，通常因为引入了外部不可控因素造成测试的不稳定性，我们不清楚问题出在哪里。另一方面其外部依赖可能会导致单测很慢。Martin Fowler建议，对于很稳定、不会引入不可控因素、速度较快的外部依赖可以存在在单测之中。针对这种情况，通常会使用mocking的方法来做单测。  
  
第三种情况，测试的方法依赖框架，在单测运行时框架代码并不会被执行。当此API在单元测试中不能被执行，我们可以假设其API工作正常，通过测试代码模拟其API执行的情况, 验证被测试对象的行为。例如在前端的单测中，单测无法执行方法中的$.ajax()方法，为了测试success、error、beforeSend、complete函数的行为，需要stub一个fake的ajax方法来达到目的。  
  
为了处理外部依赖或者框架依赖，我们引入了很多方法，比如：mocking, stubbing, dummy objects, faking objects等。这些方法的区别在于：  
  
* Dummy 对象创建后并不会被真的执行。通常用来填充参数列表。  
* Fake 对象有可以作用的实现，但通常会采用一些不适合生产环境的捷径来实现（比如说，内存数据库）。  
* Stubs 在测试中为调用提供提前准备好的结果。在测试中，通常在提前准备好的结果外不会响应其他情况。可能会记录请求信息。  
* Mocks 对象被提前按照特定的期待编写，来构成对于调用得到所期待结果的一个规范。  
  
Mocks通常用于行为测试，而Stub通常用于状态测试。Stubs的流程通常为：配置，测试，验证状态，清理。而Mocks通常为：配置数据，设置期待，测试，验证期待，验证状态，清理。  
  
通常，stubs按照stub的接口或者类提前声明了stub类，设置了特定的响应，用于测试，而mocks会在单测现场配置，设置期待。 
  
对于外部依赖，通常我们会使用Mocking的方法。在Java中，我们可以使用[Mockito框架](http://mockito.org/)来做mocking。     
  
Mockito不能够对final类、内部类、内置类型做mocking。我们使用Mockito框架来修改上面的java代码的单测代码，如下所示：  
  
```java
public class IDCheckingServiceTest {
    @Test
    public void isValidPersonTest() {
        IdService testIdService = Mockito.mock(IdService.class);
        testIdService.when(testIdService.getNameByIDNumber("23417819931221232X")
              .thenReturn("谢小萌").when(testIdService
              .getNameByIDNumber("34423319930918343x")).thenReturn("郝大龙")
              .when(testIdService.getNameByIDNumber("14010619930613762x"))
              .thenReturn("李大锤");
        IdCheckingService idCheckingService = new IdCheckingService();
        idCheckingService.setIdService(idService);
        
        assertTrue(isValidPerson("谢小萌", "23417819931221232X"));
        assertTrue(isValidPerson("郝大龙", "34423319930918343x"));
        assertTrue(isValidPerson("李大锤", "14010619930613762x"));
        assertFalse(isValidPerson(null, "14312419920707123x"));
        
        assertTrue()Mockito.verify(testIdService, Mockito.times(3));
    }
}

```    
  
## 代码变异测试（Mutation Test） 
“变异”指的是修改一处代码来改变代码行为（当然保证语法的合理性）。代码变异测试先试着对代码产生变异，然后运行单元测试，并检查是否有任何测试因为这个代码变异而失败。如果有测试失败，那么说明这个变异被“消灭”了，这是我们期望看到的结果。如果没有测试失败，则说明这个变异“存活”了下来，这种情况下我们就需要去研究一下“为什么”了。  
  
TDD讲究的是：我们应该以最简单的代码通过测试。如果基于这个假设，认可代码的变异都应该会改变代码的行为而导致测试失败。如果没有导致测试失败，要么是代码有冗余，要么是测试不足以发现这个变异。  
  
而我们对自动化测试的期望正是防止任何错误的代码修改，以减少代码维护、扩展带来的风险。而错误代码修改正是一种变异。代码变异测试帮助我们找到一些无法被当前测试防止的潜在错误。  
  
通常，许多代码的变异不会引发测试覆盖率的变化，却可能因为测试的疏漏而让错误逍遥法外。
  
常见的变异方法有：条件边界变异，反向条件变异，数学运算变异，负值翻转变异，内联常量变异，返回值变异，无/有返回值方法变异，构造函数变异等。  
  
代码变异测试出现很早，却没有被业界广泛接纳。一个原因是需要对每个代码变异反复运行测试。代码变异测试工具会消耗大量时间。单元测试可能是唯一符合代码变异测试要求的一种测试。  
  

# One more thing, tucao~
在Stash中，开发者向项目主分支merge了代码，上线后发现发生了运行时错误甚至是编译错误。开发者觉得，已经pull过最新的代码跑过单测。但这可能并无法保证代码merge后不会有问题。可能会有这种情况：  
  
![](http://7xiub6.com1.z0.glb.clouddn.com/blog/utest/mergemerge.png)       
  
正确的做法应当是在PR与merge之前都要pull最新的master分支代码到本地，运行单测，本地启动服务测试（有必要）对应API。否则可能merge之后会产生编译、运行时错误而浑然不知。  
  
然而，这样的事情本不应该交给开发人员的自觉，Stash系统应该提供这样的保证，保证在不会有可以预见到的错误时候任然允许merge，污染master分支代码，甚至可能有更加严重的后果。  
  
Atlassian公司的Stash系统提供了SDK帮助开发者开发插件。其中，[Merge Request Check Plugin Module](https://developer.atlassian.com/stash/docs/latest/reference/plugin-module-types/merge-check.html) 可以在代码merge之前做检查。通过这个接口，我们可以在merge代码到master之前，触发CI系统执行当前分支代码与master最新代码merge到本地后的单测动作，当这个任务执行完毕并且结果返回正常后，再允许merge该分支到master分支。  
  
另外，当前灰度发布的更新可以先merge到固定的dev分支，用dev做灰度测试。在灰度没有问题之后，再merge到master分支。而灰度发现问题后，这样的方式方便回滚代码，即在master分支上重新checkout新的dev分支即可。  
  
# 更多阅读
  
* [测试覆盖（率）到底有什么用？](http://www.infoq.com/cn/articles/test-coverage-rate-role)  
* [*Unit Test* by Martin Fowler](http://martinfowler.com/bliki/UnitTest.html)               
* [使用Spock框架进行单元测试](http://blog.jobbole.com/89874/) Spock是Java/Groovy上的一个新的单元测试框架。Spock有如下的特性：  
	* 可以应用于java或groovy应用的单元测试框架。  
	* 测试代码使用基于groovy语言扩展而成的规范说明语言（specification language）。  
	* 通过junit runner调用测试，兼容绝大部分junit的运行场景（ide，构建工具，持续集成等）。  
	* 框架的设计思路参考了JUnit，jMock，RSpec，Groovy，Scala，Vulcans……  
  
  
  
