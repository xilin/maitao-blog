title: JavaScriptCore oc2js的限制

date: 2019-04-30 17:07:40

author: xi.lin

categories:

- 源码学习

tags:

- JavaScriptCore
- iOS

------

# JavaScriptCore中字典数据的限制

## 问题背景

最近在写热更新需求时遇到了一个奇怪的问题，简化的场景如下（同stackoverflow上的这个[问题](https://stackoverflow.com/questions/55453128/the-entry-disappeared-in-nsdictionary-returning-from-jscontext)）：

```objective-c
    // 预定义好JSContext中的方法
    JSContext *context = [[JSContext alloc] init];
    context[@"directReturn"] = ^id(NSString *name) {
        id obj = @{@(2): @"test", @"testKey": @"testValue"};
        return obj;
    };
    // 在js中调用获得数据
    JSValue *jsValue = [context evaluateScript:@"directReturn()"];
    obj = jsValue.toObject;
    NSLog(@"jscore: %@", obj);
```

预期的输出结果应该是包含两个key的字典对象，但实际上输出的结果却是

```objective-c
jscore: {
    testKey = testValue;
}
```

`@(2): @"test"`以数字为key的字典项消失了。

<!-- more -->

## 原因分析

在*知识小集*群里求助时，有同学提到了[使用NSUserDefaults存储字典的一个坑](https://mp.weixin.qq.com/s/UKkCHdWLIpfRUqF5FEzarg)，里面提到`NSUserDefaults`作存储时必须使用`property list objects`，而只有`string objects`为key的字典才属于这类对象。从表现来看有点类似，但是从`JavaScriptCore`官方文档里找不到类似的说明做支撑。

好在`JavaScriptCore`是开源的！这种时候就只能从源码实现上来找答案了。

源码的调试环境配置可以参考之前的这篇[文章](https://tech.maitao.com/2018/11/20/build-webkit-and-chromium/)。为了简化调试流程我利用了JavaScriptCore项目里的testapi target，修改测试用例为上述问题场景。

```objective-c
static void checkNegativeNSIntegers()
{
    JSContext *context = [[JSContext alloc] init];
    context[@"test"] = ^id{
        NSDictionary *dic = @{@(1):@"v1", @"2":@"v2"};
        return dic;
    };
    JSValue *value = [context evaluateScript:@"test()"];
    NSDictionary *dic = [value toDictionary];
    NSLog(@"%@", dic);
}
```

这次的问题涉及两次数据转换。一次是从oc定义转换到js里的对象，一次是从js里的结果对象返回到oc层。

一开始我怀疑是在js返回oc对象时被过滤了，所以在`containerValueToObject`里下了断点，发现在取propertyName的时候就已经没有数字的那个key了。![js2oc](https://ws2.sinaimg.cn/large/006tNc79gy1g2ktnig765j30x50i5gt4.jpg)

那么问题应该就是出在oc对象转换到js这一层的时候。跟踪`evaluateScript:`方法，发现原因就是在`JSValue.mm`这个文件的`objectToValue`方法中，在1045行上，只有`[key isKindOfClass:[NSString class]]`时，才会存下字典项。

![oc2js](https://ws2.sinaimg.cn/large/006tNc79gy1g2ktqpq6jmj31090htwn9.jpg)

## 依据

原先在JavaScript中的实践中，有过使用数字等非字符串类型变量作key的经验，比如:

```javascript
var obj = {
  1: 'v1'
}
```

这样的对象是可以生成的。所以想当然的以为在`JavaScriptCore`中也可以这样使用。然而根据[语言规范](https://ecma-international.org/ecma-262/6.0/#sec-object-initializer-static-semantics-propname)，对象的`PropertyName`其实是做了一层隐式转换。

```
LiteralPropertyName : NumericLiteral
1. Let nbr be the result of forming the value of the NumericLiteral.
2. Return ToString(nbr).
```

查看前述js对象obj的key类型会发现

```javascript
> typeof obj[1]
< "string"
```

所以对比规范`JavaScriptCore`少了隐式转换这一层实现，选择了只保留string为key的字典项。

那么在JSPatch里又是怎么保证字典数据的完整传递呢？因为bang哥引入了`JPBoxing`，字典对象是被wrap的，所以不会有`objectToValue`时的过滤。