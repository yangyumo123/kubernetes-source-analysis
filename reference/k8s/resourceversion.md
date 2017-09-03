resourceversion分析
=================================================================
## 简介
k8s利用"resource version"的概念来实现最优并发。所有对象的metadata都有一个"resourceVersion"字段。resourceVersion表示对象的内部版本，当即将更新一个记录时，它的版本要和预先保存的版本进行检查，如果不匹配，则更新失败，返回StatusConflict（HTTP状态码409）。

每次修改对象，server都会修改resourceVersion。如果PUT操作中包含resourceVersion，那么系统将在read/modify/write周期中验证没有其他成功资源的冲突，通过验证当前resourceVersion值匹配指定的值。

目前，resourceVersion是etcd的modifiedIndex实现的。尽管如此，应用不应该考虑k8s的版本控制系统的细节。未来，可能修改resourceVersion的实现，例如使用时间戳或每个对象计数器。

client获得resourceVersion的期望值的唯一方法是从server接收它，以响应先前操作，一般是GET。该值必须被client视为不透明，并将未修改的值返回给server。客户端不应该假设资源版本在名称空间、不同类型的资源或不同的服务器上有意义。目前，resourceVersion的值被设置去匹配etcd的序列器。您可以认为它是一个逻辑时钟，API服务器可以使用它来订购请求。然而，我们期望资源版本的实现在将来会发生变化，比如我们用kind和/或命名空间来改变状态，或者到另一个存储系统的端口。

在发生冲突的情况下，正确的客户端操作是再次GET资源，重新请求更改，并尝试再次提交。这种机制可以用来防止类似以下的竞争:

    Client #1                                 Client #2
    GET Foo                                   GET Foo
    Set Foo.Bar = "one"                       Set Foo.Baz = "two"
    PUT Foo                                   PUT Foo

当这些序列并行发生时，到Foo.Bar或Foo.Baz的改变可能会丢失。

另一方面，当指定resourceVersion时，其中一个设置将会失败，因为无论哪个写成功，都会改变Foo的resourceVersion。
未来，resourceVersion可能用于其他操作的先决条件（例如，GET、DELETE），例如，存在缓存时保持读写一致性。
watch操作使用一个query parameter来指定resourceVersion。它用于指定开始监视指定资源的点。这可以用来确保在GET资源(或资源列表)和随后的watch之间没有发生任何突变，即使当前版本的资源是最近的。这是目前列表操作(GET on a collection)返回resourceVersion的主要原因。

