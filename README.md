# runtime-preliminary-study
OC runtime机制全面学习一下



```
// -------------------------------------------
// Myclass.h

#import <Foundation/Foundation.h>

@interface MyClass : NSObject<NSCopying,NSCoding>

@property (nonatomic,strong) NSArray  *array;
@property (nonatomic,  copy) NSString *string;

-(void) method1;
-(void) method2;
+(void) classMethod1;
@end

// -------------------------------------------
// Myclass.m

#import "MyClass.h"
 
@interface MyClass () {
    NSInteger       _instance1;
 
    NSString    *   _instance2;
}
 
@property (nonatomic, assign) NSUInteger integer;
 
- (void)method3WithArg1:(NSInteger)arg1 arg2:(NSString *)arg2;
 
@end
 
@implementation MyClass
 
+ (void)classMethod1 {
 
}
 
- (void)method1 {
    NSLog(@"call method method1");
}
 
- (void)method2 {
 
}
 
- (void)method3WithArg1:(NSInteger)arg1 arg2:(NSString *)arg2 
{
    NSLog(@"arg1 : %ld, arg2 : %@", arg1, arg2);
} 
@end

```

```
//
//  main.m
//  runtime01
//
//  Created by haogaoming on 2017/2/15.
//  Copyright © 2017年 郝高明. All rights reserved.
//

#import <Foundation/Foundation.h>
#import "MyClass.h"
#import <objc/runtime.h>
#import "MyClass+Tracking.h"

void TestMetaClass(id self, SEL _cmd) {
    NSLog(@"this object is %p",self);
    NSLog(@"Class id %@,super class is %@",[self class],[self superclass]);
    
    Class currentClass = [self class];
    for (int i=0; i<4; i++) {
        NSLog(@"Following the isa pointer %d times gives %p",i,currentClass);
        currentClass = objc_getClass((__bridge void *)currentClass);
    }
    
    
    NSLog(@"NSObject is class is %p",[NSObject class]);
    NSLog(@"NSObject is meta class is %p",objc_getClass((__bridge void *)[NSObject class]));
}

void imp_submethod1(id self,SEL _cmd,NSNumber* index) {
    NSLog(@"run sub method %ld",(long)index.integerValue);
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        Class newClass = objc_allocateClassPair([NSError class], "TestClass", 0);
        class_addMethod(newClass, @selector(testMetaClass), (IMP)TestMetaClass, "v@:");
        objc_registerClassPair(newClass);
    
        id instance = [[newClass alloc] initWithDomain:@"some domain" code:0 userInfo:nil];
        [instance performSelector:@selector(testMetaClass)];
    
        NSLog(@"=================== runtime对象实践 ============================");
        // runtime对象实践
        MyClass *myclass = [[MyClass alloc]init];
        //[myclass description];
        //[myclass performSelector:@selector(methodTest)];
        unsigned int outCount = 0;
    
        Class cls = myclass.class;
    
        //类名
        NSLog(@"class name is %s",class_getName(cls));
    
        NSLog(@"=============================================");
    
        //父类名字
        NSLog(@"super calss name: %s",class_getName(class_getSuperclass(cls)));
    
        NSLog(@"==============================================");
        
        //是否是元类
        NSLog(@"MyClass is %@ a meta-class",class_isMetaClass(cls) ? @"" : @"not");
        
        NSLog(@"==============================================");
        
        //获取元类
        Class meta_class = objc_getMetaClass(class_getName(cls));
        NSLog(@"%s's meta-class is %s",class_getName(cls),class_getName(meta_class));
        
        NSLog(@"==============================================");
        
        //变量实例大小
        NSLog(@"instance size : %zu",class_getInstanceSize(cls));
        
        NSLog(@"==============================================");
        
        //打印成员变量
        Ivar *ivars = class_copyIvarList(cls, &outCount);
        for (int i=0; i<outCount; i++) {
            Ivar ivar = ivars[i];
            NSLog(@"instance variable's name : %s at index: %d",ivar_getName(ivar),i);
        }
        free(ivars);
        
        NSLog(@"==============================================");

        //获取指定的成员变量
        Ivar string = class_getInstanceVariable(cls, "_string");
        if (string != NULL) {
            NSLog(@"instace variable %s",ivar_getName(string));
        }
        
        NSLog(@"==============================================");
        
        //属性操作
        objc_property_t *properties = class_copyPropertyList(cls, &outCount);
        for (int i=0; i<outCount; i++) {
            objc_property_t property = properties[i];
            NSLog(@"property's name : %s",property_getName(property));
        }
        free(properties);
        
        NSLog(@"==============================================");
        
        //获取指定的属性
        objc_property_t array = class_getProperty(cls, "array");
        if (array != NULL) {
            NSLog(@"property %s",property_getName(array));
        }
        
        NSLog(@"==============================================");
        
        //方法操作
        Method *methods = class_copyMethodList(cls, &outCount);
        for (int i=0; i<outCount; i++) {
            Method method = methods[i];
            NSLog(@"method's signature : %s",method_getName(method));
        }
        free(methods);
        
        NSLog(@"==============================================");
        
        //获取实例方法
        Method method1 = class_getInstanceMethod(cls, @selector(method1));
        if (method1 != NULL) {
            NSLog(@"method is : %s",method_getName(method1));
        }
        
        NSLog(@"==============================================");
        
        //获取类方法
        Method method2 = class_getClassMethod(cls, @selector(classMethod1));
        if (method2) {
            NSLog(@"class method is : %s",method_getName(method2));
        }
        
        NSLog(@"==============================================");
        
        //打印方法的具体实现--查看方法的实现
        IMP imp = class_getMethodImplementation(cls, @selector(method1));
        imp();
        
        NSLog(@"==============================================");
        
        //协议
        Protocol * __unsafe_unretained *protocols = class_copyProtocolList(cls, &outCount);
        Protocol *protocol;
        for (int i=0; i<outCount; i++) {
            protocol = protocols[i];
            NSLog(@"protocol name is : %s",protocol_getName(protocol));
        }
        
        NSLog(@"==============================================");
        
        Class clss = objc_allocateClassPair(myclass.class, "MySubClass", 0);
        //添加方法
        class_addMethod(clss, @selector(method3), (IMP)imp_submethod1, "v@:");
        //替换方法
        class_replaceMethod(clss, @selector(method1), (IMP)imp_submethod1, "v@:");
        //添加属性
        //属性类型  name值：T  value：变化
        //编码类型  name值：C(copy) &(strong) W(weak) 空(assign) 等 value：无
        //非/原子性 name值：空(atomic) N(Nonatomic)  value：无
        //变量名称  name值：V  value：变化
        objc_property_attribute_t type = {"T","@\"NSString\""};
        objc_property_attribute_t ownership = {"C",""};
        objc_property_attribute_t backingvar = {"N",""};
        objc_property_attribute_t attrs[] = {type,ownership,backingvar};
        class_addProperty(clss, "property2", attrs, 3);
        
        objc_registerClassPair(clss);
        
        id mySubClass = [[clss alloc] init];
        [mySubClass performSelector:@selector(method3) withObject:@(123)];
        [mySubClass performSelector:@selector(method1) withObject:@(321)];
        
        objc_property_t *mysubClassIvar = class_copyPropertyList(clss, &outCount);
        for (int i=0; i<outCount; i++) {
            objc_property_t ivar = mysubClassIvar[i];
            NSLog(@"MySubClass's property is : %s",property_getName(ivar));
        }
        free(mysubClassIvar);
        
        NSLog(@"=========================================");
        
    }
    return 0;
}
```

运行结果为：

```
2017-01-22 19:41:37.452 RuntimeTest[3189:156810] class name: MyClass
2017-01-22 19:41:37.453 RuntimeTest[3189:156810] ==========================================================
2017-01-22 19:41:37.454 RuntimeTest[3189:156810] super class name: NSObject
2017-01-22 19:41:37.454 RuntimeTest[3189:156810] ==========================================================
2017-01-22 19:41:37.454 RuntimeTest[3189:156810] MyClass is not a meta-class
2017-01-22 19:41:37.454 RuntimeTest[3189:156810] ==========================================================
2017-01-22 19:41:37.454 RuntimeTest[3189:156810] MyClass's meta-class is MyClass
2017-01-22 19:41:37.455 RuntimeTest[3189:156810] ==========================================================
2017-01-22 19:41:37.455 RuntimeTest[3189:156810] instance size: 48
2017-01-22 19:41:37.455 RuntimeTest[3189:156810] ==========================================================
2017-01-22 19:41:37.455 RuntimeTest[3189:156810] instance variable's name: _instance1 at index: 0
2017-01-22 19:41:37.455 RuntimeTest[3189:156810] instance variable's name: _instance2 at index: 1
2017-01-22 19:41:37.455 RuntimeTest[3189:156810] instance variable's name: _array at index: 2
2017-01-22 19:41:37.455 RuntimeTest[3189:156810] instance variable's name: _string at index: 3
2017-01-22 19:41:37.463 RuntimeTest[3189:156810] instance variable's name: _integer at index: 4
2017-01-22 19:41:37.463 RuntimeTest[3189:156810] instace variable _string
2017-01-22 19:41:37.463 RuntimeTest[3189:156810] ==========================================================
2017-01-22 19:41:37.463 RuntimeTest[3189:156810] property's name: array
2017-01-22 19:41:37.463 RuntimeTest[3189:156810] property's name: string
2017-01-22 19:41:37.464 RuntimeTest[3189:156810] property's name: integer
2017-01-22 19:41:37.464 RuntimeTest[3189:156810] property array
2017-01-22 19:41:37.464 RuntimeTest[3189:156810] ==========================================================
2017-01-22 19:41:37.464 RuntimeTest[3189:156810] method's signature: method1
2017-01-22 19:41:37.464 RuntimeTest[3189:156810] method's signature: method2
2017-01-22 19:41:37.464 RuntimeTest[3189:156810] method's signature: method3WithArg1:arg2:
2017-01-22 19:41:37.465 RuntimeTest[3189:156810] method's signature: integer
2017-01-22 19:41:37.465 RuntimeTest[3189:156810] method's signature: setInteger:
2017-01-22 19:41:37.465 RuntimeTest[3189:156810] method's signature: array
2017-01-22 19:41:37.465 RuntimeTest[3189:156810] method's signature: string
2017-01-22 19:41:37.465 RuntimeTest[3189:156810] method's signature: setString:
2017-01-22 19:41:37.465 RuntimeTest[3189:156810] method's signature: setArray:
2017-01-22 19:41:37.466 RuntimeTest[3189:156810] method's signature: .cxx_destruct
2017-01-22 19:41:37.466 RuntimeTest[3189:156810] method method1
2017-01-22 19:41:37.466 RuntimeTest[3189:156810] class method : classMethod1
2017-01-22 19:41:37.466 RuntimeTest[3189:156810] MyClass is responsd to selector: method3WithArg1:arg2:
2017-01-22 19:41:37.467 RuntimeTest[3189:156810] call method method1
2017-01-22 19:41:37.467 RuntimeTest[3189:156810] ==========================================================
2017-01-22 19:41:37.467 RuntimeTest[3189:156810] protocol name: NSCopying
2017-01-22 19:41:37.467 RuntimeTest[3189:156810] protocol name: NSCoding
2017-01-22 19:41:37.467 RuntimeTest[3189:156810] MyClass is responsed to protocol NSCoding
2017-01-22 19:41:37.468 RuntimeTest[3189:156810] ==========================================================

```
