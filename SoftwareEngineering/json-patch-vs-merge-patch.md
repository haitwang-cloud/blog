
>  [JSON Patch and JSON Merge Patch](https://erosb.github.io/post/json-patch-vs-merge-patch/) 的中文翻译版本。

Partly as a side effect of the `PATCH` HTTP verb gaining attention in the recent years, people started to come up with ideas about representing JSON-driven PATCH formats which declaratively describe differences between two JSON documents. The number or home-grew solutions is probably countless, two formats have been published by IETF as RFC documents to solve this problem: [RFC 6902 (JSON Patch)](https://tools.ietf.org/html/rfc6902) and [RFC 7396 (JSON Merge Patch)](https://tools.ietf.org/html/rfc7386). Both have advantages and disadvantages, and none of them will fit everybody’s usecases, so lets have a quick look at which one to use.

作为近年来受到关注HTTP的 “PATCH” 方法的带动，人们开始提出关于表示 JSON 驱动的 PATCH 格式的想法，它以声明方式描述了两个 JSON 文档之间的差异。 目前已经有许多解决方案被提出，不计其数。而其中IETF 已经发布了两种格式作为 RFC 文档来解决这个问题：[RFC 6902 (JSON Patch)](https://tools.ietf.org/html/rfc6902) 和 [RFC 7396 (JSON Merge Patch)](https://tools.ietf.org/html/rfc7386)。两者都有优缺点，会根据不同的用例因人而异，所以让我们快速比较，看看你会使用哪一个。

JSON Patch
----------

The JSON Patch format is similar to a database transaction: it is an array of mutating operations on a JSON document, which is executed atomically by a proper implementation. It is basically a series of `"add"`, `"remove"`, `"replace"`, `"move"` and `"copy"` operations.

JSON Patch format 和数据库事务类似：它是JSON 文档上的更改操作，由适当的实现自动执行。它基本上是一系列 `"add"`, `"remove"`, `"replace"`, `"move"` 和 `"copy"` 操作。


As a short example lets consider the following JSON document:

让我们通过一个简单的例子来看看JSON 文档：

```json
{
	"users" : [
		{ "name" : "Alice" , "email" : "alice@example.org" },
		{ "name" : "Bob" , "email" : "bob@example.org" }
	]
}
```

We can run the following patch on it, which changes Alice’s email address then adds a new element to the array:

我们可以通过PATCH操作更新它，这次操作更改了Alice的邮件地址，然后添加了一个新的元素到数组中：

```json
[
	{
		"op" : "replace" ,
		"path" : "/users/0/email" ,
		"value" : "alice@wonderland.org"
	},
	{
		"op" : "add" ,
		"path" : "/users/-" ,
		"value" : {
			"name" : "Christine",
			"email" : "christine@example.org"
		}
	}
]
```

The result will be:

然后结果会变成:

```json
{
	"users" : [
		{ "name" : "Alice" , "email" : "alice@wonderland.org" },
		{ "name" : "Bob" , "email" : "bob@example.org" },
		{ "name" : "Christine" , "email" : "christine@example.org" }
	]
}
```

So the outline of the operations described in a JSON Patch is

*   the `"op"` key denotes operation
*   the arguments of the operation are described by the other keys
*   there is always a `"path"` argument, which is JSON Pointer pointing to the document fragment which is the target of the operation

所以 JSON Patch 中的操作大纲是：
*  `"op"` 代表的操作类型
* 由其他键描述的操作参数
*  `"path"` 参数指向操作的目标文档片段



An interesting option of the JSON Patch specification is its `"test"` operator: its evaluation doesn’t come with any side effects, so it isn’t a data manipulating operator. Instead it can be used to describe assertions on the document at given points of the JSON Patch execution. If the `"test"` evaluates to false then an error occurs, subsequent operations won’t be executed, and the document is rolled back to its initial state. I think the `"test"` can be useful for checking preconditions before a patch execution or may be a safety net to check at the end of execution if everything looks all right. Patches are run atomically by implementations therefore if a `"test"` finds inconsistency in the document then you can safely assume that the document is still in consistent (initial) state after patch failure.

一个有趣的选项是 JSON Patch 的 `"test"` 操作符：它的执行不会产生任何副作用，所以它不是一个数据操作符。而是用来描述文档的断言，如果 `"test"` 返回 false，那么会发生错误，后续的操作不会执行，文档会回到初始状态。我觉得 `"test"` 对于在补丁执行之前检查先决条件很有用，用来检查执行后的文档是否正常。JSON Patch 是一个原子性的操作，所以如果 `"test"` 发现文档不一致，那么你可以认为文档已经回到初始状态。

JSON Merge Patch
----------------

Alongside JSON Patch there is an other JSON-based format, [JSON Merge Patch - RFC 7386](https://tools.ietf.org/html/rfc7386) , which can be used more or less for the same purpose, ie. it describes a changed version of a JSON document. The conceptual difference compared to JSON Patch is that JSON Merge Patch is similar to a diff file. It simply contains the nodes of the document which should be different after execution.

除了 JSON Patch 之外，还有另一种 JSON 格式，[JSON Merge Patch - RFC 7386](https://tools.ietf.org/html/rfc7386)，也可以用于相同的目的。**JSON Merge Patch 和 JSON Patch 的主要区别是，JSON Merge Patch 类似于 diff 文件**，它只包含文档中应该变化的节点。

As a quick example ([taken from the spec](https://tools.ietf.org/html/rfc7386#section-1)) if we have the following document:

请看一个简单的例子[数据源](https://tools.ietf.org/html/rfc7386#section-1)，假设我们有以下文档：


```json
{
	"a": "b",
	"c": {
		"d": "e",
		"f": "g"
	}
}
```

Then we can run the following patch on it:

我们可以通过下面的内容patch它

```json
{
	"a":"z",
	"c": {
		"f": null
	}
}
```

which will change the value of `"a"` to `"z"` and will delete the `"f"` key.

它将改变 `"a"` 的值为 `"z"`，并将 `"f"` 键删除。

The simplicity of the format may look first promising at the first glance, since most probably anyone understanding the schema of the original document will also instantly understand a merge patch document too. It is just a standardization of one may naturally call a patch of a JSON document.

乍一看，这个格式可能会看起来很更简便一些，因为大多数人都能够直接理解原始文档的格式。因为很可能任何了解原始文档架构的人也会立即理解merge patch文档。 它只是一个标准化的一个可以自然地称为一个 JSON 文档的PATCH。


But this simplicity comes with some limitations:

*   Deletion happens by setting a key to `null`. This inherently means that it isn’t possible to change a key’s value to `null`, since such modification cannot be described by a merge patch document.
*   Arrays cannot be manipulated by merge patches. If you want to add an element to an array, or mutate any of its elements then you have to include the entire array in the merge patch document, even if the actually changed parts is minimal.
*   the execution of a merge patch document never results in error. Any malformed patch will be merged, so it is a very liberal format. It is not necessarily good, since you will probably need to perform programmatic check after merge, or run a JSON Schema validation after the merge.

但这种简单性也有一些限制：
*  删除是通过设置键的值为 `null` 来实现的。这意味着不能通过改变键的值来删除键，因为这种改变不能被merge patch描述。
*  数组不能被merge patch操作。如果你想添加一个元素到数组，或者改变数组中的任何元素，那么你必须包含整个数组在merge patch文档中，即使是微小的改动。
* merge patch不会导致错误。任何不正确的patch都会被合并，所以它是一个很严格的格式。它不一定是好的，因为你可能需要在合并后进行程序性检查，或者在合并后运行JSON Schema验证。


Summary
-------

JSON Merge Patch is a naively simple format, with limited usability. Probably it is a good choice if you are building something small, with very simple JSON Schema, but you want offer a quickly understandable, more or less working method for clients to update JSON documents. A REST API designed for public consumption but without appropriate client libraries might be a good example.

JSON Merge Patch是一个简易的patch操作，但是它也有明显的局限性。如果你正在建造一个小的项目，使用简单的JSON Schema，它可能是一个很好的选择。但你希望提供一个可以被客户端读取的快速易懂，较为可用的方法来更新JSON文档。面向公共消费而设计但没有适当的客户端库的 REST API 可能是一个更好的选择。

For more complex usecases I’d pick JSON Patch, since it is applicable to any JSON documents (unline merge patch, which is not able to set keys to `null`). The specification also ensures atomic execution and robust error reporting.

对于更加复杂的使用场景，我会偏向于JSON Patch，因为它适用于任何JSON文档（不论是merge patch，它不能设置键为 `null`）。该规范也保证了原子执行和robust的错误报告。
