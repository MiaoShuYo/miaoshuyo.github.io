我们可以在文档中存储任何数据内容，比如在订单文档中我们会存储订单状态、订单物品数量、订单金额等等内容。但是我们还需要存储一些和订单文档无关的内容，比如谁修改了订单文档、什么时候修改了订单文档等，这时就需要 Document Metadata （文档元数据，我们暂且这样翻译）登场了 。
#### Metadata 默认存储什么
Metadata 的存储格式和文档本身一样也是 Json，RavenDB 使用 Metadata 存储有关跟踪文档的几个重要信息：
1. 集合名称，存储在 **@collection** 中，通过这个属性何以确定数据文档存储在哪个集合中，如果该值未设置，数据文档将存储在 **@empty** 集合中；
2. 文档最后修改日期，存储在 **@last-modified** 属性中，存储格式时 UTC；
3. 客户端类型，这时一个 Key ，我们可以通过这个 Key 得知客户端的类型，常见类型如下表：

| 类型|说明|
|---|---|
|Raven-Clr-Type|.NET客户端|
|Raven-Java-Class|Java 客户端|
|Raven-Python-Class|Python客户端|

#### 自定义 Metadata 属性命名规范
除了使用 RavenDB 内置的 Metadata 属性外我们还可以自定义 Metadata 属性，比如我们要记录订单文档最后的修改人是谁，那么我们可以自定义 Metadata 属性 **Last-Modified-By-User**，代码如下：
```csharp
using Raven.Client.Documents;

var store = new DocumentStore
{
    Urls = new[] { "http://localhost:8080" },
    Database = "Tasks"
};
store.Initialize();
using (var session = store.OpenSession())
{
    var order = session.Load<Order>("orders/1-A");
    //这里不会再次请求服务端，因为在我们查询数据文档时，
    //Metadata 也会跟着一起返回给客户端
    var metadata = session.Advanced.GetMetadataFor(order);
    metadata["Last-Modified-By-User"] = "张三";
    session.SaveChanges();
}
```
我们在 RavenDB Studio 中查看 **orders/1-A** 数据内容，我们可以看到自定一的 Metdata 属性已经存在与 Metadata 节点下了，如下图。
![在这里插入图片描述](https://img-blog.csdnimg.cn/3ff36c76630f431fb81236ab78c2f328.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5Za15Y-U5ZOf,size_19,color_FFFFFF,t_70,g_se,x_16)
一般来说在实际开发中我们很少使用这种形式来操作 Metadata ，我们会使用事件的方式来操作，这将在后面的专栏里具体讲解，在这里你只需知道当前我们所讲的即可。

>TIP：当我们在 RavenDB 文档中看到以 **@** 开头的 Metadata 属性时，就说明这个属性是 RavenDB 保留给自己用的，因此我们在扩展 Metadata 属性时不能使用与之一样的属性名，我们自定义的 Metadata 属性需遵循帕斯卡命名法（PascalCase）或者Pascal-Case命名法（组成属性名的单词首字母大写，单词之间用 **-** 号分割），当然你也可以不按照这个建议。