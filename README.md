# JMH 框架初步

## 缘起

+ 开发组同学使用SimpleDateFormat普遍都是都是随用随new
+ 发现网上搜到的关于SimpleDateFormat性能分析的说明都很烂
  - 大多说new 一个SimpleDateFormat代价很高，高在哪儿没有实例证明
  - 和线程安全混着说的，有点关系，但是其实是两个维度
  - 测试代码很初级，jvm性能测试要考虑的问题实际上很多，所谓的微基准其实更复杂

然后就想起有这么个玩意儿，没正经用过

## 起步

这个东西其实很官方，所以直接移步官网[tutorial](https://openjdk.java.net/projects/code-tools/jmh/)

如介绍，起步姿势很容易，需要预先有什么也很显然：

```
mvn archetype:generate \
          -DinteractiveMode=false \
          -DarchetypeGroupId=org.openjdk.jmh \
          -DarchetypeArtifactId=jmh-java-benchmark-archetype \
          -DgroupId=me.eleizh.mbenchmark.dateformat \
          -DarchetypeVersion=1.21 \
          -DartifactId=dfmb \
          -Dversion=1.0
```

上面改成自己工程的groupId和artifactId，我这里比官网的多一个archetypeVersion到最新版的原型，对应也是JMH的最新版，但是后面可能还得改pom，因为它生成的缺省是java 1.6，我们至少1.8吧

这个例子里面是12，买新不买旧么

其实这时候什么都不写就能跑，不是很科学的感觉，因为它MyBenchmark类里面有个被@Benchmark注解的空方法，假么假是的就跑起来了

# 撰写点内容

这个基于一开始的目的确实不复杂

+ 跟写JUnit差不多
+ 加上注解@Benchmark的方法就是要测的东西
+ 有一些初始化要在setUp里面做并且配合@State注解，这主要是ThreadLocal这个字段是独立于测试过程之外的，但它事实上又和测试息息相关，所以要受框架管理，所以你就看人家专业人士写的东西就是考虑的全面
+ @Stage注解必须给一个参数，这个见最下面的参考连接解释，目前我这也没看的太清楚

# 跟着大家一起喊，别猜，测试！

就是怎么run起来的问题

```
mvn clean package && java -jar target/benchmark.jar
```
但其实呢，最好打完包之后先别急着跑，用java -jar target/benchmark.jar -h 看一下文档，看看可以设置的参数的缺省值，当然我也是假设，确实写这个框架的人家就是牛，测试的次数，预热的时间等等都是有根据的

# 回到起点

这里简单说一下跑这组对比测试的结论：

+ 测试环境是一个vsphere管理的vm，4c8g，宿主机是大路货R730，CPU是E5-2620 v4
+ 虚机系统是centos 7.6，还跑了不少玩意儿在上面，但free+cache的内存怎么也有4g，负载很低
+ java -jar target/benchmark.jar，什么都不加，缺省跑出来的是吞吐量指标，随用随new大概50w上下的样子，而共享的是200w左右的样子，他还有一个Error值估计在吞吐量是指的delta值，所以差距还是很明显的，如下：
```
# Run complete. Total time: 00:16:43

REMEMBER: The numbers below are just data. To gain reusable insights, you need to follow up on
why the numbers are the way they are. Use profilers (see -prof, -lprof), design factorial
experiments, perform baseline and negative tests that provide experimental control, make sure
the benchmarking environment is safe on JVM/OS/HW level, ask for reviews from the domain experts.
Do not assume the numbers tell you what you want them to tell.

Benchmark                                          Mode  Cnt        Score       Error  Units
MyBenchmark.testSimpleDateFormatEveryNew          thrpt   25   550059.182 ±  5090.087  ops/s
MyBenchmark.testSimpleDateFormatEveryThreadLocal  thrpt   25  2035301.197 ± 18864.674  ops/s
```
+ 但是如果```java -jar target/benchmark.jar -bm sample```是统计的样本的响应时间，每个操作消耗的时间，这个就没那么明显了，这个直接copy下来某次的执行结果看着玩儿了
+ 所以再没跑更多测试的情况下可以说，稍具规模的负载下避免每次new一个实例是有价值的，可以显著的提升吞吐量

```
# Run complete. Total time: 00:16:44

REMEMBER: The numbers below are just data. To gain reusable insights, you need to follow up on
why the numbers are the way they are. Use profilers (see -prof, -lprof), design factorial
experiments, perform baseline and negative tests that provide experimental control, make sure
the benchmarking environment is safe on JVM/OS/HW level, ask for reviews from the domain experts.
Do not assume the numbers tell you what you want them to tell.

Benchmark                                                                                        Mode      Cnt   Score    Error  Units
MyBenchmark.testSimpleDateFormatEveryNew                                                       sample  8587010  ≈ 10⁻⁶            s/op
MyBenchmark.testSimpleDateFormatEveryNew:testSimpleDateFormatEveryNew·p0.00                    sample           ≈ 10⁻⁶            s/op
MyBenchmark.testSimpleDateFormatEveryNew:testSimpleDateFormatEveryNew·p0.50                    sample           ≈ 10⁻⁶            s/op
MyBenchmark.testSimpleDateFormatEveryNew:testSimpleDateFormatEveryNew·p0.90                    sample           ≈ 10⁻⁶            s/op
MyBenchmark.testSimpleDateFormatEveryNew:testSimpleDateFormatEveryNew·p0.95                    sample           ≈ 10⁻⁶            s/op
MyBenchmark.testSimpleDateFormatEveryNew:testSimpleDateFormatEveryNew·p0.99                    sample           ≈ 10⁻⁶            s/op
MyBenchmark.testSimpleDateFormatEveryNew:testSimpleDateFormatEveryNew·p0.999                   sample           ≈ 10⁻⁵            s/op
MyBenchmark.testSimpleDateFormatEveryNew:testSimpleDateFormatEveryNew·p0.9999                  sample           ≈ 10⁻⁵            s/op
MyBenchmark.testSimpleDateFormatEveryNew:testSimpleDateFormatEveryNew·p1.00                    sample            0.008            s/op
MyBenchmark.testSimpleDateFormatEveryThreadLocal                                               sample  7596485  ≈ 10⁻⁶            s/op
MyBenchmark.testSimpleDateFormatEveryThreadLocal:testSimpleDateFormatEveryThreadLocal·p0.00    sample           ≈ 10⁻⁶            s/op
MyBenchmark.testSimpleDateFormatEveryThreadLocal:testSimpleDateFormatEveryThreadLocal·p0.50    sample           ≈ 10⁻⁶            s/op
MyBenchmark.testSimpleDateFormatEveryThreadLocal:testSimpleDateFormatEveryThreadLocal·p0.90    sample           ≈ 10⁻⁶            s/op
MyBenchmark.testSimpleDateFormatEveryThreadLocal:testSimpleDateFormatEveryThreadLocal·p0.95    sample           ≈ 10⁻⁶            s/op
MyBenchmark.testSimpleDateFormatEveryThreadLocal:testSimpleDateFormatEveryThreadLocal·p0.99    sample           ≈ 10⁻⁶            s/op
MyBenchmark.testSimpleDateFormatEveryThreadLocal:testSimpleDateFormatEveryThreadLocal·p0.999   sample           ≈ 10⁻⁵            s/op
MyBenchmark.testSimpleDateFormatEveryThreadLocal:testSimpleDateFormatEveryThreadLocal·p0.9999  sample           ≈ 10⁻⁵            s/op
MyBenchmark.testSimpleDateFormatEveryThreadLocal:testSimpleDateFormatEveryThreadLocal·p1.00    sample            0.006            s/op
```

如果做更多维度的测试，应该会更有意思，特别是考虑gc因素还有运行时虚拟参数的时候，尽情体现微基准的乐趣吧

## 参考

+ [Stack Overflow上的一个问题及高票答案](https://stackoverflow.com/questions/504103/how-do-i-write-a-correct-micro-benchmark-in-java)，这里还提到其他一些微基准框架
+ [中文搬运工一枚](https://benjaminwhx.com/2018/06/15/%E4%BD%BF%E7%94%A8JMH%E5%81%9A%E5%9F%BA%E5%87%86%E6%B5%8B%E8%AF%95/)，有intellij里面配置的方法，还有一些细节注意事项，值得好好参考
+ [中文更高级搬运工一枚](https://www.cnkirito.moe/java-jmh/)，提供了不少原理性的参考信息
+ [参考多多益善](https://cloud.tencent.com/developer/article/1119316)，这个内容不多，但是写的很流畅

## 结语

没有测量，没有发言权，让玄学和猜测让位给理性和逻辑；值得深入搞一搞的玩意儿
