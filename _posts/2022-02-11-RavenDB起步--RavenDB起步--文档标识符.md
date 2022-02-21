在关系型数据库中表一般情况下都会存在主键，这个主键在所在表中是唯一的不可重复的，同样在 RavenDB 中也存在这样的主键，它被成为文档标识符或文档ID。文档ID是由 UTF8 字符串组成的最多 2025 字节长度的全局唯一值。一般来说文档 ID 的组成规则是： **集合名称** + **/** + **唯一值** ，当然如果你有其他文档 ID 组成的规则也可以使用。下面我们来看一下 RavenDB 生成文档 ID 的策略。
>TIP：RavenDB 的文档 ID 是数据库全局唯一的，这和关系型数据库的主键是所在表唯一不一样。另外文档 ID 建议使用可读性高的字符串，例如：User/Order/001 。

#### 一、语义（外部）文档 ID
生成文档 ID 最常见的方式是用户生成文档 ID，它经常用在生成具有一定意义的 ID 的情况下。实现代码如下：
```csharp
using Raven.Client.Documents;

var store = new DocumentStore
{
    Urls = new[] { "http://localhost:8080" },
    Database = "Tasks"
};
store.Initialize();
using (var session=store.OpenSession())
{
    var person = new Person
    {
        Name = "张三"
    };
    session.Store(person,"person/zhangsan@demo.com");
    session.SaveChanges();
}
```
```session.Store(person,"person/zhangsan@demo.com");```该行代码将文档 ID 设置为 email 地址，这样我们就可以根据用户输入的电子邮件轻松定位到文档。

#### 二、嵌套文档 ID 
嵌套文档 ID 是语义文档 ID 的一个特例，比如在一个大型电商系统中，每个 User 都有可能存在多个订单，那么如果我们将每个 User 的所有订单都放在 User 文档中显然是不合理的，正确的做法应该是将订单单独作为一个文档存储。那么这时订单文档的文档 ID 的生成策略就应该是：**User文档 ID + / + Order 文档ID**。
#### 三、客户端生成文档 ID (hilo)
在大部分情况下，我们不希望考虑文档 ID 的生成策略，希望由 RavenDB 来帮我们生成文档 ID。在 RavenDB 中我们可以使用hilo，在我们第一次需要生成 ID 时，向服务器请求保留文档 ID 范围，这时服务器将会确保所提供的范围只对一个客户端使用，然后我们的客户端就可以在给定的范围内安全的生成文档 ID。这种方法是目前最好的生成文档 ID 的方法，可以保证在客户端非常繁忙的情况下扩展，并快速生成大量文档 ID。
这里存在一个问题，当又多个客户端从不同的节点同时向服务端发送保留文档 ID 的请求时，很有可能出现这几个客户端获得打文档 ID 范文是一样的，那么为了解决这个问题各个节点之间会相互通信，如果发现有节点的文档 ID 范围和自己的一样的话，这些节点所生成的文档 ID 后面将会加上节点名称，比如 A 节点和 B 节点同时请求服务端保留文档 ID 范围，这时它俩的文档 ID 范围都是 100-150，那么这两个节点所生成的文档 ID 将会是类似于 User/100-A 和 User/100-B 这样的。
#### 四、服务器端生成文档 ID
虽然 Hilo 可以生成可读性和可预测性较好的文档 ID，但是它需要客户端和服务端合作才能使用，但是如果我们需要手动在 RavenDB Studio 中编写文档或者只指定文档 ID 开头的情况呢？这时我们就可以使用服务端生成文档 ID 策略。比如我们要在 RavenDB Studio中创建一个订单数据，这时我们在 ID 中输入 ***order/*** 然后单击 Save ， RavenDB 就会为我们自动生成一个类似于下图的文档 ID。
![在这里插入图片描述](https://img-blog.csdnimg.cn/91e47690ad1c4aa4a4807b1a3911b2a9.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5Za15Y-U5ZOf,size_20,color_FFFFFF,t_70,g_se,x_16)
同样，使用代码时我们可以在指定文档 ID 时只指定文档 ID 的开头，代码如下：
```csharp
using Raven.Client.Documents;

var store = new DocumentStore
{
    Urls = new[] { "http://localhost:8080" },
    Database = "Tasks"
};
store.Initialize();
using (var session=store.OpenSession())
{
    var person = new Person
    {
        Name = "张三"
    };
    session.Store(person, "user/");
    session.SaveChanges();
}

```
运行下面代码后，在 RavenDB Studio 中查看 person 文档，新添加的数据如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/f70bb1751862415184110a2949b5ea7c.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5Za15Y-U5ZOf,size_20,color_FFFFFF,t_70,g_se,x_16)
#### 五、Identity 生成文档 ID 策略
如果在开发中需要生成连续的文档 ID ，那么我们可以使用 Identity 生成文档 ID 策略。Identity 每次生成文档 ID 都需要向服务器发送请求。Identity 生成文档 ID 和服务器端生成文档 ID 很相似，但是不是使用 / 而是使用 | 来作为结尾生成 ID。我们在  RavenDB Studio 的 ID 中输入：**order|** 即可使用 Identity 生成 ID。生成结果如下图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/57af23a255b543a19f6e999f550dea91.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5Za15Y-U5ZOf,size_20,color_FFFFFF,t_70,g_se,x_16)
这个策略存在一个问题，如果我们尝试使用 ID 保存文档并且保存失败，值仍然会递增。因此，即使 Identity 生成连续数字，如果事务已回滚，它仍可能会跳过标识符。
同时在分布式环境中，这种策略需要我们防止竞争，例如两个客户端在两个不同的服务器上生成相同的 Identity ，生成新 Identity 的部分过程需要节点相互协调。这意味着我们需要通过网络与集群中的其他成员通信，以确保我们拥有 Identity 的下一个值。这会增加保存带有 Identity 的新文档的成本。更糟糕的是，在故障情况下，我们可能无法与集群中足够数量的节点进行通信。这意味着我们也将无法生成请求的 Identity。
>TIP：除非有明确的要求必须使用连续的 ID，否则这种策略不予考虑。
#### 六、总结
我们已经讨论了很多生成文档标识符的选项，每个选项都有自己的行为和成本，各种方法之间也存在性能差异。
RavenDB 通过将文档 ID 存储在 B+Tree 中来跟踪它们。如果文档 ID 非常大，则意味着 RavenDB 可以在给定空间中存储更少的文档 ID。
1. hilo 生成的文档 ID 在词法上可排序，在大多数情况下，我们可以获得优质的树和非常有效的搜索，并且它还生成最易读的内容；
2. 使用斜线的服务器端方法在存储适用性方面最佳值。比 hilo 生成的 ID 要大一些，但就底层存储而言，它是按词法排序的，以及时可预测的，从而弥补了这一点。它非常适合大型批处理作业，并且在其中包含许多额外的优化；
3. 语义生成的 ID 是未排序的，RavenDB 可以轻松处理大量带有语义标识符的文档，对于性能来说也没什么大问题；
4. Identity 生成文档 ID ，需要网络请求才能生成下一个值，如果节点无法与集群中的大多数节点通信，这个方法将变得不可用。