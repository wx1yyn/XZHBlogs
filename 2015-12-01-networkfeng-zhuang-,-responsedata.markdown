---
layout: post
title: "Network封装、ResponseData"
date: 2015-12-01 10:40:08 +0800
comments: true
categories: 
---

***

###服务器返回的 响应数据 中的 `每一个小的数据项` 的抽象 `ResponseData 实体类`

- 整个响应数据，也是看做一个 `数据项`，由其他更多的 小数据项 组成
- 而响应数据中的每一个子key对于的value，又被看做是一个 `数据项`

***

###看如下一个JSON结构

```objc
{
	"code" : 200,
	"data" :{

		//字符串value
		"name" : "xiongzenghui",

		//整形数值value
		"age" : 19,

		//bool value
		"isVip" : 0,

		//数组value
		"favorates" : ["打篮球", "看电影", "爬山"],

		//字典value
		"key" : {

			"key1" : "value1",
			"key2" : "value2",
			"key3" : "value3",
		}
	},
}
```

###上述JSON解析成 `ResponseData 实体类对象`

![](http://i8.tietuku.com/af0d0882aa89e518.png)

###最终从ResponseData实例，取出对应基本数据类型的值

- NSString 
- NSNumber
- BOOL
- NSDate

###对于 NSDictionary与NSArray类型则继续解析，直到所有只有`基本数据类型`的值

***

###服务器响应数据的解析格式

```objc
typedef NS_ENUM(NSInteger, ResponseDataFormat) {
    
    //1. json
    ResponseDataFormatJson = 0,
    
    //2. xml
    ResponseDataFormatXml,
    
    //3. string
    ResponseDataFormatString
    
    //4. 还可以扩展其他，如:image，plist..
    //...
    
};
```

###每一个 数据项 的数据类型

```objc
typedef NS_ENUM(NSInteger, XZHResponseDataType) {
    
    //未知
    ResponseDataUnknown = 0,
    
    //没有数据
    ResponseDataNone,
    
    //有数据，只是为null
    ResponseDataNull,
    
    ResponseDataNumber,
    ResponseDataString,
    ResponseDataArray,
    ResponseDataDictionary
} ;
```

###ResponseData.h

```objc
/**
 *  一个响应数据项(字段)的抽象
 */
@interface XZHResponseData : NSObject

//////////////////////////////////////////////////////////////////////////
////////XZHResponseData实例初始化
//////////////////////////////////////////////////////////////////////////

/**
 *  传入response data初始化
 *  并得到response data的数据类型
 */
- (id)initWithData:(id)data;

/**
 * Response data
 */
@property (nonatomic, strong) id data;

/**
 *  网络应答的数据类型
 */
@property(nonatomic, readonly) XZHResponseDataType dataType;


//////////////////////////////////////////////////////////////////////////
////////data是基本数据类型
//////////////////////////////////////////////////////////////////////////

/**
 *  是否允许存在null类型字段值
 */
@property(nonatomic, assign) BOOL isAllowNull;

/**
 *  整个response data 否为空值
 *  1. 不存在值
 *  2. 存在之，但是为null
 */
- (BOOL)isNull;

/**
 *  指定key数据是否为空值
 *  1. 只针对字典
 *  2. 字典key对应value没有值 或 null
 */
- (BOOL)isNullForKey:(NSString *)key;

/**
 * 获取指定key的数据类型
 */
- (XZHResponseDataType)typeForKey:(NSString*)key;

/**
 *	转成字符串
 */
- (NSString *)stringValue;
- (NSString *)stringValue:(NSError *__autoreleasing*)error;

/**
 *  转成数字类型
 */
- (NSNumber *)numberValue;
- (NSNumber *)numberValue:(NSError *__autoreleasing*)error;

/**
 *	转成bool类型
 */
- (BOOL)boolValue;
- (BOOL)boolValue:(NSError **)error;

/**
 *	转成时间戳
 */
- (NSDate *)timestampValue;
- (NSDate *)timestampValue:(NSError *__autoreleasing*)error;

/**
 *	转成日期格式的NSDate
 */
- (NSDate *)dateTimeValue:(NSDateFormatter *)formatter;
- (NSDate *)dateTimeValue:(NSDateFormatter *)formatter error:(NSError *__autoreleasing*)error;


//////////////////////////////////////////////////////////////////////////
////////data是一个JSON字典
//////////////////////////////////////////////////////////////////////////
- (XZHResponseData *)childDataForKey:(NSString *)key error:(NSError *__autoreleasing *)error;


//////////////////////////////////////////////////////////////////////////
////////data是一个Array数组
//////////////////////////////////////////////////////////////////////////


- (NSString *)stringValueForIndex:(NSInteger)index;

- (NSNumber *)numberValueForIndex:(NSInteger)index;

- (BOOL)boolValueForIndex:(NSInteger)index;

- (NSDate *)timestampValueForIndex:(NSInteger)index;

- (NSDate *)dateTimeValueForIndex:(NSInteger)index DataFormatter:(NSDateFormatter *)formatter;

- (XZHResponseData *)childDataForIndex:(NSInteger)index;

/**
 *	当前数据项的 子key 的个数
 */
- (NSUInteger)count;

/**
 *  获取当前responsedata的所有key
 */
- (NSArray *)allKeys;

@end
```

传入的data类型如下三种:


- 基本数据类型: 直接可以取出基本数据类型的值
	
- JSON字典: 继续封装成一个ResponseData对象

- Array数组: 取出index对应的data
	- sub data 依然可以为如下三种形式
		- 基本数据类型
		- JSON字典
		- Array数组


***

###传入data实例化ResponseData对象时，识别出data的数据类型并保存

```objc
- (id)initWithData:(id)data
{
    self = [super init];
    if (self)
    {
        //保存数据项
        _data = data;

        //默认数据项可以为NULL
        _isAllowNull = YES;
        
        //得到数据项的数据类型
        if (data == nil)
            _dataType = XZHResponseDataTypeNone;
        else if ([data isKindOfClass:[NSNull class]])
            _dataType = XZHResponseDataTypeNull;
        else if ([data isKindOfClass:[NSString class]])
            _dataType = XZHResponseDataTypeString;
        else if ([data isKindOfClass:[NSNumber class]])
            _dataType = XZHResponseDataTypeNumber;
        else if ([data isKindOfClass:[NSArray class]])
            _dataType = XZHResponseDataTypeArray;
        else if ([data isKindOfClass:[NSDictionary class]])
            _dataType = XZHResponseDataTypeDictionary;
        else
            _dataType = XZHResponseDataTypeUnknown;
    }
    
    return self;
}
```

###获取当前数据项ResponseData中的 子数据项的 数据类型

```objc
- (XZHResponseDataType)typeForKey:(NSString *)key {
    
    //1. sub key 对应的 data
    id value = [_data objectForKey:key];
    
    //2. 创建ResponseData实例，计算数据类型
    XZHResponseData *data = [[XZHResponseData alloc] initWithData:value];
    
    return data.dataType;
}
```

###当前数据项是否为`空`

```objc
- (BOOL)isNull {
    return _dataType == XZHResponseDataTypeNull || _dataType == XZHResponseDataTypeNone;
}
```

###子数据项是否为`空`

```objc
- (BOOL)isNullForKey:(NSString *)key {
    
    if (!key || [key isEqualToString:@""]) {
        return YES;
    }
    
    //必须是字典才可以取key
    if (_dataType != XZHResponseDataTypeDictionary) {
        return YES;
    }
    
    id value = [_data objectForKey:key];
    
    if (!value || [value isKindOfClass:[NSNull class]]) {
        return YES;
    } else {
        return NO;
    }
}
```

###列举一个数据项是基本数据类型`字符串`时，获取其对应的value

```objc
- (NSString *)stringValue {
    return [self stringValue:nil];
}

- (NSString *)stringValue:(NSError *__autoreleasing *)error_p {
    
    //为NULL，并且该数据项允许为NULL，转换成nil，防止崩溃
    if ([self isNull] && _isAllowNull) {
        return nil;
    }
    
    //数据项类型是 `字符串`
    if (self.dataType == XZHResponseDataTypeString) {
        return self.data;
    }
    
    //数据项类型是 `数值`
    if (self.dataType == XZHResponseDataTypeNumber) {
        return [self.data stringValue];
    }
    
    //不是如上两种类型的数据不可以转化成String
    *error_p = [self _typeConvertError];
    
    return nil;
}

- (NSString *)stringValueForIndex:(NSInteger)index {
    XZHResponseData *data = [self childDataForIndex:index];
    if (data) {
        return [data stringValue];
    } else {
        return nil;
    }
}
```

###如果子数据项又是一个JSON字典

```objc
- (XZHResponseData *)childDataForKey:(NSString *)key error:(NSError *__autoreleasing *)error {
    
    //必须是字典类型
    if (_dataType != XZHResponseDataTypeDictionary) {
        *error = [self _typeConvertError];
        return nil;
    }
    
    //key不能为空
    if ([key xIsNullOrEmpty]) {
        *error = [self _typeConvertError];
        return nil;
    }
    
    return [[XZHResponseData alloc] initWithData:[_data objectForKey:key]];
}

- (XZHResponseData *)childDataForIndex:(NSInteger)index {
    
    if (_dataType != XZHResponseDataTypeArray) {
        return nil;
    }
    
    id data = [_data xSafe_objectAtIndex:index];
    
    return [[XZHResponseData alloc] initWithData:data];
}
```

套路就跟JSON解析实体类差不多.

