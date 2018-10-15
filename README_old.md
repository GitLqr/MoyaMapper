![MoyaMapper](https://github.com/LinXunFeng/MoyaMapper/raw/master/Screenshots/MoyaMapper.png)



<center>

[![](https://img.shields.io/badge/author-LinXunFeng-blue.svg)](https://cocoapods.org/pods/MoyaMapper)[![CI Status](https://img.shields.io/travis/LinXunFeng/MoyaMapper.svg?style=flat)](https://travis-ci.org/LinXunFeng/MoyaMapper)[![Version](https://img.shields.io/cocoapods/v/MoyaMapper.svg?style=flat)](https://cocoapods.org/pods/MoyaMapper)[![License](https://img.shields.io/github/license/LinXunFeng/MoyaMapper.svg)](https://cocoapods.org/pods/MoyaMapper)[![Platform](https://img.shields.io/cocoapods/p/MoyaMapper.svg?style=flat)](https://cocoapods.org/pods/MoyaMapper)

</center>

MoyaMapper是基于Moya和SwiftyJSON封装的工具，以Moya的plugin的方式来实现间接解析，支持RxSwift

## Usage

### 一、注入插件 

1. 定义一个遵守ModelableParameterType协议的结构体

```swift
// 各参数返回的内容请参考下面JSON数据对照图
struct NetParameter : ModelableParameterType {
    static var successValue: String { return "false" }
    static var statusCodeKey: String { return "error" }
    static var tipStrKey: String { return "" }
    static var modelKey: String { return "results" }
}
```

此外，这里还可以做简单的路径处理，以应付各种情况，以'>'隔开

```swift
// 如：
error: {
    'errorStatus':false
    'errMsg':'error Argument type'
}

static var tipStrKey: String { return "error>errMsg" }
```



2. 以plugin的方式传递给MoyaProvider

```swift
let lxfNetTool = MoyaProvider<LXFNetworkTool>(plugins: [MoyaMapperPlugin(NetParameter.self)])
```

### 二、定义模型

1. 创建一个遵守Modelable协议的结构体

```swift
struct MyModel: Modelable {
    
    var _id : String
    var address : AddressModel
    ......
    
    init?(_ json: JSON) {
        self._id = json["_id"].stringValue
        self.address = json["address"].modelValue(AddressModel.self)
        ......
    }
}

struct AddressModel: Modelable {
    
    var street : String
    ......
    
    init(_ json: JSON) {
        self.street = json["street"].stringValue
        ......
    }
}
```



2. 模型嵌套解析

使用泛型的方式为 `JSON` 类拓展两个解析方法，使解析模型嵌套易如反掌

```swift
// 解析单个模型
modelValue<T: Modelable>(_ type: T.Type) -> T

// 解析数组模型
modelsValue<T: Modelable>(_ type: T.Type) -> [T]
```

示例代码：

```
self.address = json["address"].modelValue(AddressModel.self)
```



### 三、解析

这里只贴出主要代码

- Normal

```swift
lxfNetTool.request(.data(type: .all, size: 10, index: 1)) { result in
    guard let response = result.value else { return }
    
    // Models
    let models = response.mapArray(MyModel.self)
    for model in models {
        print("id -- \(model._id)")
    }
    
    // Result
    let (isSuccess, tipStr) = response.mapResult()
    print("isSuccess -- \(isSuccess)")
    print("tipStr -- \(tipStr)")
    
    // Model
    /*
    let model = response.mapObjResult(MyModel.self)
    */
    
    // 获取指定路径的值
    // response.fetchJSONString(keys: []])
    // response.fetchJSONString(path: "", keys: [])
    // response.fetchString(path: "", keys: [])
    
    // 使用自定义模型参数类
    /*
    let (result, models) = response.mapArrayResult(MyModel.self, params: { () -> (ModelableParameterType.Type) in
        return CustomParameter.self
    })
     */
    
}
```

- Rx

```swift
// let rxRequest: Single<Response>

// MARK: Rx
let rxRequest = lxfNetTool.rx.request(.data(type: .all, size: 10, index: 1))

// Models
rxRequest.mapArray(MyModel.self).subscribe(onSuccess: { models in
    for model in models {
        print("id -- \(model._id)")
    }
}).disposed(by: dispseBag)

// Models + Result
rxRequest.mapArrayResult(MyModel.self).subscribe(onSuccess: { (result, models) in
    print("isSuccess --\(result.0)")
    print("tipStr --\(result.1)")
    print("models count -- \(models.count)")
}).disposed(by: dispseBag)

// 获取指定路径的值
rxRequest.fetchString(keys: [0, "_id"]).subscribe(onSuccess: { str in
    // 取第1条数据中的'_id'字段对应的值
    print("str -- \(str)")
}).disposed(by: dispseBag)
```

<hr>

### JSON数据对照

为方便理解，这里给出具体使用`JSON数据图`，结合 `Example`食用更佳～

- 单层模型

![JSON数据对照-单层模型](https://github.com/LinXunFeng/MoyaMapper/raw/master/Screenshots/JSON数据对照-单层模型.png)

- 模型嵌套

![JSON数据对照-模型嵌套](https://github.com/LinXunFeng/MoyaMapper/raw/master/Screenshots/JSON数据对照-模型嵌套.png)





### 返回类型注释：

- result

```swift
// static var successValue: String { return "false" }
// static var statusCodeKey: String { return "error" }
// static var tipStrKey: String { return "" }

// 元祖类型
// 参数1：根据statusCodeKey取出的值与successValue是否相等
// 参数2：根据tipStrKey取出的值
result：(Bool, String)
```

- fetchString

```swift
// fetchJSONString(keys: <[JSONSubscriptType]>)
1、通过 keys 传递数组, 该数组可传入的类型为 Int 和 String
2、默认是以 modelKey 所示路径，来获取相应的数值。如果modelKey非你所要用的起始路径，可以使用下方的重载方法重新指定路径

// response.fetchJSONString(path: <String?>, keys: <[JSONSubscriptType]>)
```



## Misc

### 自定义Response

在无网状态下使用Moya发送请求时，response为nil，则插件式注入无效化，可以在 `catchError`方法中自定义`response`并返回。使用方法如下所示：

```swift
rxRequest.catchError { (error) -> PrimitiveSequence<SingleTrait, Response> in
    // 捕获请求失败（如：无网状态），自定义response
    let err = error as NSError
    let resBodyDict = ["error":"true", "errMsg":err.localizedDescription]
    let response = Response(resBodyDict, statusCode: 203, parameterType: NetParameter.self)
    return Single.just(response)
}.mapArrayResult(MyModel.self).subscribe(onSuccess: { (result, models) in
    print("isSuccess --\(result.0)")
    print("tipStr --\(result.1)")
    print("models count -- \(models.count)")
}).disposed(by: dispseBag)
```



**配合 `MoyaSugar` 使用更加方便**



## CocoaPods

- 默认安装

MoyaMapper默认只安装Core下的文件

```ruby
pod 'MoyaMapper'
```



- RxSwift拓展

```ruby
pod 'MoyaMapper/Rx'
```



## License

MoyaMapper is available under the MIT license. See the LICENSE file for more info.



## Author

LinXunFeng

- email: 598600855@qq.com
- Blogs
  - [linxunfeng.top](http://linxunfeng.top/)
  - [掘金](https://juejin.im/user/58f8065e61ff4b006646c72d/posts)
  -  [简书](https://www.jianshu.com/u/31e85e7a22a2)