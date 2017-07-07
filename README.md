# **runtime 源码讲解**   
runtime  就是运行时机制 ，可以动态的增加一个类 给类添加属性 方法或者是新建一个类
或者是改变类的实现方法

oc 其实就是一种运行时语言 和 c++ 不同是的c++在编译的时候 就已经 直接把函数地址写进可执行文件 。而oc是在程序运行的时候 runtime根据条件作出判断 --函数标志和
函数过程的真正内容之间可以动态的修改

下面是对runtime源码的一些解读 
****
```
typedef struct objc_class *Class;
struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;

#if !__OBJC2__
    Class super_class   指向父类的指针                                  
    const char *name        类名                                  
    long version            类版本号                                  
    long info                                                
    long instance_size         对象的占用空间                           
    struct objc_ivar_list *ivars      所有的属性                       
    struct objc_method_list **methodLists     所有的 方法              
    struct objc_cache *cache                    调用的方法 会储存在这个里面 如果方便下次直接调用 。方便下次直接调用  不用再找一遍 是通过 
    标志符号 SEL 查找的            
    struct objc_protocol_list *protocols      遵守的协议               
#endif

} OBJC2_UNAVAILABLE;
```

这就是oc中类的定义  class 就是指向 objc_class 的结构体的一个指针 objc_class结构体    包含了类的一些基本的信息 
```
struct objc_ivar_list *ivars 
struct objc_ivar_list {
    int ivar_count  属性的数量                                          
#ifdef __LP64__
    int space            占用空间的大小                                      
#endif
    /* variable length structure */
    struct objc_ivar ivar_list[1]    结构体  objc_ivar（其实就是属性）的数组                        
} 
```
ivars 其实就是指向了这样的一个结构体 的指针  

```
struct objc_ivar ivar_list[1] 
struct objc_ivar {
    char *ivar_name    属性的名字                                      OBJC2_UNAVAILABLE;
    char *ivar_type   属性的类型                                         OBJC2_UNAVAILABLE;
    int ivar_offset         地址偏移字节                                  OBJC2_UNAVAILABLE;
#ifdef __LP64__
    int space         空间大小                                       OBJC2_UNAVAILABLE;
#endif
} 
```
结构体 就是这个 类的属性

```
struct objc_method_list {
    struct objc_method_list *obsolete                         

    int method_count      方法数量                                    
#ifdef __LP64__
    int space              占用的空间                                  
#endif
    /* variable length structure */
    struct objc_method method_list[1]        objc_method 数组               
}     
```
这个结构体存储 了类的方法列表

下面是一个方法的定义  
```
struct objc_method {
    SEL method_name         sel类型标识符                                  OBJC2_UNAVAILABLE;
    char *method_types          方法类型 参数 和 返回值类型                             
    IMP method_imp               具体的函数指针                            OBJC2_UNAVAILABLE;
} 

SEL 是指向这个结构体的指针 
typedef struct objc_selector *SEL;
typedef id (*IMP)(id, SEL, ...);  一个函数指针

```
这就是oc中一个方法 的定义  因为 类中的方法 
******
## **runtime方法介绍**
****

###  **object_getClass(id obj)**   



```
/** 
 * 返回对象的类的isa 指针
 * 
 * @param obj 你想检查的对象
 * 
 * @return 返回这个对象的isa指针 , 
 *  or  Nil if  object is nil.
 */
OBJC_EXPORT Class object_getClass(id obj) 
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0);
```

这个方法是返回  对象的isa指针   的方法   对 调用的是对象的时候 和【objc class】返回的class 是一样的都是isa 指针 当 调用的是类对象的时候  object_getClass(id obj) 返回的是类对象的isa 指针 。指向的是元类对象 。 类对象调用【objc class】返回的是自己本身


###     **object_setClass(id obj,Class cls)**   
```
/** 
 * 给对象设置一个类
 * 
 * @param obj The object to modify.
 * @param cls A class object.
 * 
 * @return The previous value of \e object's class, or \c Nil if \e object is \c nil.
 */
OBJC_EXPORT Class object_setClass(id obj, Class cls) 
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0);
```
给一个对象 设置成别的类
```
 person *people = [[person alloc]init];
        
        people.name = @"张文勇";
        
        [people class];
        
        //设置类
        object_setClass(people, [dog class]);
        
        获取对象所属的类
        NSLog(@"==%@==",object_getClass(people));
```

###  **object_isClass(id obj)**   

```
/** 
 * 判断一个对象是否是类对象
 *
 * Returns whether an object is a class object.
 * 
 * @param obj An Objective-C object.
 * 
 * @return true if the object is a class or metaclass, false otherwise.
 */
OBJC_EXPORT BOOL object_isClass(id obj)
    OBJC_AVAILABLE(10.10, 8.0, 9.0, 1.0);
```

###  **object_getIvar(id obj,Ivar ivar)**   

```

/** 
* 读取实例对象属性的值
 * Reads the value of an instance variable in an object.
 * 
 * @param obj 对象.
 * @param ivar The Ivar describing the instance variable whose value you want to read.
 * 
 * @return 返回属性的值  如果obj是空则返回nil
 
 * 
 * @note \c object_getIvar is faster than \c object_getInstanceVariable if the Ivar
 *  for the instance variable is already known.
 */
OBJC_EXPORT id object_getIvar(id obj, Ivar ivar) 
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0);
```

这个方法 是获取对象 属性的值的方法。 举例子如下
```

 先出初始化一个对象
 person *people = [[person alloc]init];
        people.name = @"张文勇";
        通过 objec_getClass()获取这个对象的isa指针 看看属于哪个类
        然后再 获取 这个类的一个name的Ivar 属性
        Ivar  ivar = class_getInstanceVariable(object_getClass(people), "_name");
        

        // 然后通过这个属性 object_getIvar(id obj,Ivar ivar);获取属性值
        
        NSString *name =      object_getIvar(people, ivar);
    
        NSLog(@"%@",name);
```

###  **object_setIvar(id obj,Ivar ivar,id value)**   
给对象的某一个属性赋值 
```
/** 
 * 给对象的属性赋值
 * 
 * @param obj 要赋值的对象.
 * @param ivar 描述变量属性的Ivar
  The Ivar describing the instance variable whose value you want to set.
 * @param value 要赋的值
 * 
 * @note 变量属性 with known memory management (such as ARC strong and weak)
 *  use that memory management. I
 
 如果对象没有 默认 的 memory management  就使用unsafe_unretained;

 nstance variables with unknown memory management 。
 *  are assigned as if they were unsafe_unretained.
 * @note \c object_setIvar is faster than \c object_setInstanceVariable if the Ivar
 *  for the instance variable is already known.
 */
OBJC_EXPORT void object_setIvar(id obj, Ivar ivar, id value) 
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0);
```

  如果对象没有 默认 的 memory management  就使用unsafe_unretained;   
这个方方法 再ARC可用 
例子
```
person *people = [[person alloc]init];
        
       
        people.name = @"张文勇";
        Ivar  ivar = class_getInstanceVariable(object_getClass(people), "_name");
        
        
        NSString *name =      object_getIvar(people, ivar);
    
        //给对想得到某个属性赋值  给对象的某个属性赋值 
        给对象的某个属性赋值 。给对象的某个属性赋值
        object_setIvar(people, ivar, @"tianya");
        
        
        NSLog(@"%@",people.name);
```

### **bject_setIvarWithStrongDefault(id obj, Ivar ivar, id value)**

 这个方法 和上个方法 基本一致唯一的区别 就是属性如果没有默认memory management 就实用strong 类型的    。
```
/** 
 * Sets the value of an instance variable in an object.
 * 
 * @param obj The object containing the instance variable whose value you want to set.
 * @param ivar The Ivar describing the instance variable whose value you want to set.
 * @param value The new value for the instance variable.
 * 
 * @note Instance variables with known memory management (such as ARC strong and weak)
 *  use that memory management. Instance variables with unknown memory management 
 *  are assigned as if they were strong.
 * @note \c object_setIvar is faster than \c object_setInstanceVariable if the Ivar
 *  for the instance variable is already known.
 */
OBJC_EXPORT void object_setIvarWithStrongDefault(id obj, Ivar ivar, id value) 
    OBJC_AVAILABLE(10.12, 10.0, 10.0, 3.0);
```
****
#   **Obtaining Class Definitions获取类定义**   

##   **objc_getClass(const char *name)**   

 根据类的字符串的名字获取类    
```
/**
 
 * Returns the class definition of a specified class.
 * 

 * @param name The name of the class to look up.
 * 
 * @return The Class object for the named class, or \c nil
 *  if the class is not registered with the Objective-C runtime.
 * 
 * @note \c objc_getClass is different from \c objc_lookUpClass in that if the class
 *  is not registered, \c objc_getClass calls the class handler callback and then checks
 *  a second time to see whether the class is registered. \c objc_lookUpClass does 
 *  not call the class handler callback.
 * 
 * @warning Earlier implementations of this function (prior to OS X v10.0)
 *  terminate the program if the class does not exist.
 */
OBJC_EXPORT Class objc_getClass(const char *name)
    OBJC_AVAILABLE(10.0, 2.0, 9.0, 1.0);

```
 objc_getClass(const char *name) 获取类的例子   
```

        根据类的名字获取类对象 。
        Class ddd = objc_getClass("person");
        
        根据类对象 获取_name属性
        Ivar ivar=  class_getInstanceVariable(ddd, "_name");
        
        类实例化对象 
        
         id ccc =  [[ddd alloc]init];
        
        给对象 _name 属性添加 属性址
        object_setIvar(ccc, ivar, @"dwedwe");
        
        获取实例对象的属性值
        NSLog(@"%@",object_getIvar(ccc, ivar));
        
```

##     **objc_getMetaClass(const char *name)**   

     根据字符串得到 类对象的元类   


```
/** 
 * Returns the metaclass definition of a specified class.
 * 
 * @param name The name of the class to look up.
 * 
 * @return The \c Class object for the metaclass of the named class, or \c nil if the class
 *  is not registered with the Objective-C runtime.
 * 
 * @note If the definition for the named class is not registered, this function calls the class handler
 *  callback and then checks a second time to see if the class is registered. However, every class
 *  definition must have a valid metaclass definition, and so the metaclass definition is always returned,
 *  whether it’s valid or not.
 */
OBJC_EXPORT Class objc_getMetaClass(const char *name)
    OBJC_AVAILABLE(10.0, 2.0, 9.0, 1.0);
```
##  **objc_getRequiredClass**   
这个方法 和 objc_getClass(const char *name)方法一下样都是根据类的字符串的名字返回一个
类对象 。不过这个方法 如果找不到类会直接 crash    程序
```
/** 
 * Returns the class definition of a specified class.
 * 
 * @param name 需要找的类的名字 。需要找的类的名字 需要找的类的名字
 * 
 * @return 类对象.
 * 
 * @note This function is the same as \c objc_getClass, but kills the process if the class is not found.
 * @note This function is used by ZeroLink, where failing to find a class would be a compile-time link error without ZeroLink.
 */
OBJC_EXPORT Class objc_getRequiredClass(const char *name)
    OBJC_AVAILABLE(10.0, 2.0, 9.0, 1.0);
```

## **int objc_getClasssList(class *buffer,int buffercount)**
这个方法是获取注册类的总数 class *buffer是储存所有的类数组
```
/** 
 * Obtains the list of registered class definitions.
 * 
 * @param buffer An array of \c Class values. On output, each \c Class value points to
 *  one class definition, up to either \e bufferCount or the total number of registered classes,
 *  whichever is less. You can pass \c NULL to obtain the total number of registered class
 *  definitions without actually retrieving any class definitions.
 * @param bufferCount An integer value. Pass the number of pointers for which you have allocated space
 *  in \e buffer. On return, this function fills in only this number of elements. If this number is less
 *  than the number of registered classes, this function returns an arbitrary subset of the registered classes.
 * 
 * @return An integer value indicating the total number of registered classes.
 * 
 * @note The Objective-C runtime library automatically registers all the classes defined in your source code.
 *  You can create class definitions at runtime and register them with the \c objc_addClass function.
 * 
 * @warning You cannot assume that class objects you get from this function are classes that inherit from \c NSObject,
 *  so you cannot safely call any methods on such classes without detecting that the method is implemented first.
 */
OBJC_EXPORT int objc_getClassList(Class *buffer, int bufferCount)
    OBJC_AVAILABLE(10.0, 2.0, 9.0, 1.0);
```
利用这个方法 可以获取所有已经注册的类 代码实战
```
      int classcount = 0;
        
        首先获取所有的类的中数
        classcount  = objc_getClassList(NULL, 0);
        
        Class *buffbuffer1 = NULL;
        根据类的总数给数组分配空间
        buffbuffer1  = (Class *)malloc(sizeof(Class)*classcount);
        
        然后获取类的总数 并且 把所有的类存储在数组中
        objc_getClassList(buffbuffer1, classcount);
        
        
        
        for(int i=0;i<classcount;i++)
        {
            
            根据类对象获取类的名字 打印类名字
         const  char *dd = class_getName(buffbuffer1[i]);
            
            
            printf("%s\n",dd);
        }
         free(buffbuffer1);
        
        buffbuffer1 = NULL;
```
##  *objc_copyClassList(unsigned int *outcount)**   
这个方法 和上面的getClassList的方法一样 不过更简单 也是获取所有的已经住的类
```
/** 
 * Creates and returns a list of pointers to all registered class definitions.
 * 
 * @param outCount An integer pointer used to store the number of classes returned by
 *  this function in the list. It can be \c nil.
 * 
 * @return A nil terminated array of classes. It must be freed with \c free().
 * 
 * @see objc_getClassList
 */
OBJC_EXPORT Class *objc_copyClassList(unsigned int *outCount)
    OBJC_AVAILABLE(10.7, 3.1, 9.0, 1.0);
```

     代码实例   

```
  Class *buffbuffer1 = objc_copyClassList(&classcount);
        
        
        free(buffbuffer1);
```
******
#        *Working with Classes*类方法   
##     **class_getName(Class cls)**   
根据类 对象获取了类的字符串名字 
```


/** 
 * Returns the name of a class.
 * 
 * @param cls A class object.
 * 
 * @return The name of the class, or the empty string if \e cls is \c Nil.
 */
OBJC_EXPORT const char *class_getName(Class cls) 
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0);
```
代码实例 
```
 Class dd = [person class];
        class_getName(dd);
````
## **class_isMetaClass**   
判断一个类是否是元类
```
/** 
 * Returns a Boolean value that indicates whether a class object is a metaclass.
 * 
 * @param cls A class object.
 * 
 * @return \c YES if \e cls is a metaclass, \c NO if \e cls is a non-meta class, 
 *  \c NO if \e cls is \c Nil.
 */
OBJC_EXPORT BOOL class_isMetaClass(Class cls) 
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0);
```

##  **class_getSuperclass**   
返回这个类的父类
```
/** 
 * Returns the superclass of a class.
 * 
 * @param cls A class object.
 * 
 * @return The superclass of the class, or \c Nil if
 *  \e cls is a root class, or \c Nil if \e cls is \c Nil.
 *
 * @note You should usually use \c NSObject's \c superclass method instead of this function.
 */
OBJC_EXPORT Class class_getSuperclass(Class cls) 
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0);
```
代码实例
```
      Class dd  = [person class];
        
        获取赋类
       Class ddc =   class_getSuperclass(dd);
        获取 类的名字
        const char *ddda =   class_getName(ddc);
        
        printf("%s\n",ddda);
        
```
## **class_getVersion**   
获取类的版本号
```
/** 
 * Returns the version number of a class definition.
 * 
 * @param cls A pointer to a \c Class data structure. Pass
 *  the class definition for which you wish to obtain the version.
 * 
 * @return An integer indicating the version number of the class definition.
 *
 * @see class_setVersion
 */
OBJC_EXPORT int class_getVersion(Class cls)
    OBJC_AVAILABLE(10.0, 2.0, 9.0, 1.0);
```

## **class_setVersion**
```
OBJC_EXPORT void class_setVersion(Class cls, int version)
    OBJC_AVAILABLE(10.0, 2.0, 9.0, 1.0);
```
## **class_getInstanceSize**
获取实例变量的大小
```
/** 
 * Returns the size of instances of a class.
 * 
 * @param cls A class object.
 * 
 * @return The size in bytes of instances of the class \e cls, or \c 0 if \e cls is \c Nil.
 */
OBJC_EXPORT size_t class_getInstanceSize(Class cls) 
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0);
```
##  **class_getInstaceVarialble(class cls,const char *name)**   
根据属性的名字获取类的某一个属性 
```
/** 
 * Returns the \c Ivar for a specified instance variable of a given class.
 * 
 * @param cls The class whose instance variable you wish to obtain.
 * @param name The name of the instance variable definition to obtain.
 * 
 * @return A pointer to an \c Ivar data structure containing information about 
 *  the instance variable specified by \e name.
 */
OBJC_EXPORT Ivar class_getInstanceVariable(Class cls, const char *name)
    OBJC_AVAILABLE(10.0, 2.0, 9.0, 1.0);
```
##     *classes_copyIvarList(Class cls,unsigned int *outCount)**   
获取所有类的所有的属性 获取类的所有的属性 获取类的所有的属性 但是 不会获取父类的属性 只会获取 自己的属性
```
/** 
 * Describes the instance variables declared by a class.
 * 
 * @param cls 要检查.
 * @param outCount 用于返回 属性数组的长度
 * 如果outcount是NULL值则不会返回长度
 * 
 * @return An array of pointers of type Ivar describing the instance variables declared by the class. 
     从父类集成来的不回包含在内
 *  Any instance variables declared by superclasses are not included. The array contains *outCount 
 *  pointers followed by a NULL terminator. You must free the array with free().
 * 
 *  If the class declares no instance variables, or cls is Nil, NULL is returned and *outCount is 0.
 */
OBJC_EXPORT Ivar *class_copyIvarList(Class cls, unsigned int *outCount) 
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0);
```
代码实例
```
 xiaoming *zhang = [[xiaoming alloc]init];
        
        zhang.age = @"dwedw";
        
        Class xiam = [zhang class];
        
       unsigned int count  = 0;
      
      //定义一个属性数组的指针
        Ivar *list = NULL;
        ／／获取属性数组 。和数组的长度
        list =   class_copyIvarList(xiam, &count);
        for (int i=0;i<count;i++)
        {
             id huilai;
            Ivar ivar = list[i];
            获取对象 这个属性的值 。
            并且不回获取赋类的这个属性
          huilai =  object_getIvar(zhang, ivar);
             NSLog(@"%@\n",huilai);   
        }
        free(list);
```
看看这个查找的源码   
```
Ivar *class_copyIvarList(Class cls_gen, unsigned int *outCount)
{

    //根据类对象 生成一个 old_class
    struct old_class *cls = oldcls(cls_gen);
    Ivar *result = NULL;
    unsigned int count = 0;
    int i;

     
    if (!cls) {
        //如果类对象不存在 直接返回
        if (outCount) *outCount = 0;
        return NULL;
    }

    if (cls->ivars) {
        查看结构体中的 属性里俩啊吗的数量
        count = cls->ivars->ivar_count;
    }

    if (count > 0) {

         根据属性数组的数量申请内存
        result = malloc((count+1) * sizeof(Ivar));

        使用数组去接受 这些 属性
        for (i = 0; i < cls->ivars->ivar_count; i++) {
            result[i] = (Ivar)&cls->ivars->ivar_list[i];
        }
        result[i] = NULL;
    }

返回
    if (outCount) *outCount = count;
    return result;
}

```
## **class_getInstanceMethod(Class cls,SEL name)**

根据SEL获取这个类的对象的实例方法（ 包含父类   ）

```
/** 
 * Returns   返回给定类的实例方法  
 * 
 * @param cls 类对象.
 * @param name The selector of the method you want to retrieve.
 * 
 * @return The method that corresponds to the implementation of the selector specified by 
 *  \e name for the class specified by \e cls, or \c NULL if the specified class or its 
 *  superclasses do not contain an instance method with the specified selector.
 *
 * @note This function searches superclasses for implementations, whereas \c class_copyMethodList does not. 这个方法 搜索 父类
 */
OBJC_EXPORT Method class_getInstanceMethod(Class cls, SEL name)
    OBJC_AVAILABLE(10.0, 2.0, 9.0, 1.0);

```
##  **class_getClassMethod**   
获取类方法 ( 包含父类   )
```
/** 
 * Returns a pointer to the data structure describing a given class method for a given class.
 * 
 * @param cls A pointer to a class definition. Pass the class that contains the method you want to retrieve.
 * @param name A pointer of type \c SEL. Pass the selector of the method you want to retrieve.
 * 
 * @return A pointer to the \c Method data structure that corresponds to the implementation of the 
 *  selector specified by aSelector for the class specified by aClass, or NULL if the specified 
 *  class or its superclasses do not contain an instance method with the specified selector.
 
 *
 * @note Note that this function searches superclasses for implementations, 
 *  whereas \c class_copyMethodList does not.
 */
OBJC_EXPORT Method class_getClassMethod(Class cls, SEL name)
    OBJC_AVAILABLE(10.0, 2.0, 9.0, 1.0);
```

##     **class_getMethodImplementation(Class cls,SEL name)**   
根据SEL获取对应的执行此消息是 所调用的函数指针
```
/** 
 * Returns the function pointer that would be called if a 
 * particular message were sent to an instance of a class.
 * 
 * @param cls The class you want to inspect.
 * @param name A selector.
 * 
 * @return The function pointer that would be called if \c [object name] were called
 *  with an instance of the class, or \c NULL if \e cls is \c Nil.
 *
 * @note \c class_getMethodImplementation may be faster than \c method_getImplementation(class_getInstanceMethod(cls, name)).
 
 * @note The function pointer returned may be a function internal to the runtime instead of
 *  an actual method implementation. For example, if instances of the class do not respond to
 *  the selector, the function pointer returned will be part of the runtime's message forwarding machinery.
 */
OBJC_EXPORT IMP class_getMethodImplementation(Class cls, SEL name) 
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0);
```

## **class_respondsToSelector(Class cls,SEL sel)**

查看 类中有没有这个方法
```
/** 
 * Returns a Boolean value that indicates whether instances of a class respond to a particular selector.
 * 
 * @param cls The class you want to inspect.
 * @param sel A selector.
 * 
 * @return \c YES if instances of the class respond to the selector, otherwise \c NO.
 * 
 * @note You should usually use \c NSObject's \c respondsToSelector: or \c instancesRespondToSelector: 
 *  methods instead of this function.
 */
OBJC_EXPORT BOOL class_respondsToSelector(Class cls, SEL sel) 
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0);
```
##  **class_copyMethodList**   
获取类的方法列表

```
/** 
 * Describes the instance methods implemented by a class.
 * 
 * @param cls The class you want to inspect.
 * @param outCount On return, contains the length of the returned array. 
 *  If outCount is NULL, the length is not returned.
 * 
 * @return An array of pointers of type Method describing the instance methods 
 *  implemented by the class—any instance methods implemented by superclasses are not included. 
 *  The array contains *outCount pointers followed by a NULL terminator. You must free the array with free().
 * 
 *  If cls implements no instance methods, or cls is Nil, returns NULL and *outCount is 0.
 * 
 * @note To get the class methods of a class, use \c class_copyMethodList(object_getClass(cls), &count).
 * @note To get the implementations of methods that may be implemented by superclasses, 
 *  use \c class_getInstanceMethod or \c class_getClassMethod.
 */
OBJC_EXPORT Method *class_copyMethodList(Class cls, unsigned int *outCount) 
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0);
```
代码 例子
```
          
        unsigned int count = 0;
        获取所有的方法数组
        Method *dd = class_copyMethodList([dog class], &count);
        
        
        for(int i=0;i<count;i++)
        {
            Method mmm = dd[i];

            根据 Method获取SEL
           SEL seee =  method_getName(mmm);
            
            根据SEL获取方法名字
            const char *str = sel_getName(seee);
            
            printf("---%s---\n",str);
        
        }
        

```
## **class_conformsToProtocol(Class cls,Protocol *protocol)**
看某个类是否遵守某个协议(     @note You should usually use NSObject's conformsToProtocol: method instead of this function.   )
```
/** 
 * Returns a Boolean value that indicates whether a class conforms to a given protocol.
 * 
 * @param cls The class you want to inspect.
 * @param protocol A protocol.
 *
 * @return YES if cls conforms to protocol, otherwise NO.
 *
 * 
 */
OBJC_EXPORT BOOL class_conformsToProtocol(Class cls, Protocol *protocol) 
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0);
```

## **class_copyProtocolList**   
这个类所遵守的所有的协议
```
/** 
 * Describes the protocols adopted by a class.
 * 
 * @param cls The class you want to inspect.
 * @param outCount On return, contains the length of the returned array. 
 *  If outCount is NULL, the length is not returned.
 * 
 * @return An array of pointers of type Protocol* describing the protocols adopted 
 *  by the class. Any protocols adopted by superclasses or other protocols are not included. 
 *  The array contains *outCount pointers followed by a NULL terminator. You must free the array with free().
 * 
 *  If cls adopts no protocols, or cls is Nil, returns NULL and *outCount is 0.
 */
OBJC_EXPORT Protocol * __unsafe_unretained *class_copyProtocolList(Class cls, unsigned int *outCount)
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0);

```
## **class_getProperty(Class cls,const char *name)**
根据名字返回类中的声明的一个属性
```
/** 
 * Returns a property with a given name of a given class.
 * 
 * @param cls The class you want to inspect.
 * @param name The name of the property you want to inspect.
 * 
 * @return A pointer of type \c objc_property_t describing the property, or
 *  \c NULL if the class does not declare a property with that name, 
 *  or \c NULL if \e cls is \c Nil.
 */
OBJC_EXPORT objc_property_t class_getProperty(Class cls, const char *name)
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0);

```

## **class_getProperty**

## **class_copyPropertyList**
```
/** 
 * Returns a property with a given name of a given class.
 * 
 * @param cls The class you want to inspect.
 * @param name The name of the property you want to inspect.
 * 
 * @return A pointer of type \c objc_property_t describing the property, or
 *  \c NULL if the class does not declare a property with that name, 
 *  or \c NULL if \e cls is \c Nil.
 */
OBJC_EXPORT objc_property_t class_getProperty(Class cls, const char *name)
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0);

/** 
 * Describes the properties declared by a class.
 * 
 * @param cls The class you want to inspect.
 * @param outCount On return, contains the length of the returned array. 
 *  If \e outCount is \c NULL, the length is not returned.        
 * 
 * @return An array of pointers of type \c objc_property_t describing the properties 
 *  declared by the class. Any properties declared by superclasses are not included. 
 *  The array contains \c *outCount pointers followed by a \c NULL terminator. You must free the array with \c free().
 * 
 *  If \e cls declares no properties, or \e cls is \c Nil, returns \c NULL and \c *outCount is \c 0.
 */
OBJC_EXPORT objc_property_t *class_copyPropertyList(Class cls, unsigned int *outCount)
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0);
```
代码实例
```
unsigned int coutn =0;
        
        objc_property_t *das = class_copyPropertyList([person class], &coutn);
        
        
        
        NSLog(@"==%d\n",coutn);
```

##     **class_getIvarLayout**   

```
/** 
 * Returns a description of the \c Ivar layout for a given class.
 * 
 * @param cls The class to inspect.
 * 
 * @return A description of the \c Ivar layout for \e cls.
 */
OBJC_EXPORT const uint8_t *class_getIvarLayout(Class cls)
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0);
```



##   **class_addMethod(Class cls,SEL name,IMP imp,const char *types)**   

给 类增加 一个实例方法 
```
/** 
 * Adds a new method to a class with a given name and implementation.
 * 
 * @param cls The class to which to add a method.
 * @param name A selector that specifies the name of the method being added.
 * @param imp A function which is the implementation of the new method. The function must take at least two arguments—self and _cmd.
 * @param types An array of characters that describe the types of the arguments to the method. 
 * 
 * @return YES if the method was added successfully, otherwise NO 
 *  (for example, the class already contains a method implementation with that name).
 *
 * @note class_addMethod will add an override of a superclass's implementation, 
 *  but will not replace an existing implementation in this class. 
 *  To change an existing implementation, use method_setImplementation.
 */
OBJC_EXPORT BOOL class_addMethod(Class cls, SEL name, IMP imp, 
                                 const char *types) 
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0);
```
代码例子
```
         person *ss =   [[person alloc]init];
        
        
        SEL 参数随便起名字 。他只是一种标志 Method的标志 。
        IMP 函数指针 。
        IMP指针函数的定义 正如下面定义的函数一样 一点参数必须是id self ，第二个参事是SEL _cmd
        从第三个参数开始 是函数的参数 



   添加实例方法的最后一个参数是描述函数指针参数的 一个const char * 
   v代表 void 
   @ 代表对象id 
   ：代表SEL
   @代表对象NSString

    

  BOOL dd =       class_addMethod([person class], @selector(zhangwenyong), (IMP)mydd, "v@:@");
        
        
        [ss performSelector:@selector(zhangwenyong) withObject:@"好的天简好课方"];
        

```
```

void mydd(id self,SEL _cmd,NSString *str)
{
    NSLog(@"%@",str);
}
```
给类添加方法 是可行的 给类动态的添加方法 。 给类动态的添加方法
##  **class_replaceMethod(class cls,SEL name,IMP imp, const char *type)**   
 
 替换类的方法 

```
/** 
 * Replaces the implementation of a method for a given class.
 * 
 * @param cls The class you want to modify.
 * @param name A selector that identifies the method whose implementation you want to replace.
 * @param imp The new implementation for the method identified by name for the class identified by cls.
 * @param types An array of characters that describe the types of the arguments to the method. 
 *  Since the function must take at least two arguments—self and _cmd, the second and third characters
 *  must be “@:” (the first character is the return type).
 * 
 * @return The previous implementation of the method identified by \e name for the class identified by \e cls.
 * 
 * @note This function behaves in two different ways:
 *  - If the method identified by \e name does not yet exist, it is added as if \c class_addMethod were called. 
 *    The type encoding specified by \e types is used as given.
 *  - If the method identified by \e name does exist, its \c IMP is replaced as if \c method_setImplementation were called.
 *    The type encoding specified by \e types is ignored.
 */
OBJC_EXPORT IMP class_replaceMethod(Class cls, SEL name, IMP imp, 
                                    const char *types) 
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0);

```
替换类的方法 

代码实例
```
void  shangzhan(id self,SEL _cmd)
{
    NSLog(@"替换的方法");
}
        class_replaceMethod([person class], @selector(personaaa), (IMP)shangzhan, "v@:");
        
        person *pd =[[person alloc]init];
        
        [pd personaaa];
```
类的方法的替换 。这是类的方法 的替换 这是类的方法的替换
##  **class_addIvar**   
给类增加属性 。给类增加属性 。给类增加属性 给类增加属性 给类增加属性
```
/** 
 * Adds a new instance variable to a class.
 * 
 * @return YES if the instance variable was added successfully, otherwise NO 
 *         (for example, the class already contains an instance variable with that name).
 *
 * @note This function may only be called after objc_allocateClassPair and before objc_registerClassPair. 
 *       Adding an instance variable to an existing class is not supported.
 * @note The class must not be a metaclass. Adding an instance variable to a metaclass is not supported.
 * @note The instance variable's minimum alignment in bytes is 1<<align. The minimum alignment of an instance 
 *       variable depends on the ivar's type and the machine architecture. 
 *       For variables of any pointer type, pass log2(sizeof(pointer_type)).
 */
OBJC_EXPORT BOOL class_addIvar(Class cls, const char *name, size_t size, 
                               uint8_t alignment, const char *types) 
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0);

```
代码是实例
```
     
        //生成一个类
        Class animal=   objc_allocateClassPair([NSObject class], "animal", 0);
        给类增加一个属性 
        给类增加一个属性
        不能给存在生成的类用这个方法添加属性 不能给意境生成的类添加这个属性
        class_addIvar(animal, "name", sizeof(NSString *), log2(sizeof(NSString *)), @encode(NSString *));
        注册这个类 
        objc_registerClassPair(animal);
        
        ／／根据类生成对象
        id dd = [[animal alloc]init];
        ／／根据名字 获取 对象的name属性
        Ivar ivar =   class_getInstanceVariable(animal, "name");
        
      // 给这个属性赋值
        object_setIvar(dd, ivar, @"张文勇");
        

       unsigned  int count = 0;
        
        ／／ 获取这个类的所有的属性 
        ／／获取这个类的所有属性 


        Ivar *ddd =   class_copyIvarList(animal, &count);
        
        
        NSLog(@"=数量==%d==\n",count);
        
        
        Ivar wodedd = class_getInstanceVariable(animal, "name");
      
        id dpppp = object_getIvar(dd, wodedd);
        
        NSLog(@"==%@==",dpppp);
        
        
        for(int i=0;i<count;i++)
        {

           id dasasds = object_getIvar(dd, ddd[i]);
    
            NSLog(@"==变量==%@\n",dasasds);
            
        }
        
```

## **class_addProtocol**
```
** 
 * Adds a protocol to a class.
 * 
 * @param cls The class to modify.
 * @param protocol The protocol to add to \e cls.
 * 
 * @return \c YES if the method was added successfully, otherwise \c NO 
 *  (for example, the class already conforms to that protocol).
 */
OBJC_EXPORT BOOL class_addProtocol(Class cls, Protocol *protocol) 
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0);
```





**分割点**   
**********
 
##  **Adding Classes**   
##  **objc_allocateClassPair**   
这个方法用于生成 一个类 。这个方法 用于生成一个类 。
这个方法 用于生成  一个类


```
/** 
 * Creates a new class and metaclass.
 * 
 * @param superclass The class to use as the new class's superclass, or \c Nil to create a new root class.
 * @param name The string to use as the new class's name. The string will be copied.
 * @param extraBytes The number of bytes to allocate for indexed ivars at the end of 
 *  the class and metaclass objects. This should usually be \c 0.
 * 
 * @return The new class, or Nil if the class could not be created (for example, the desired name is already in use).
 * 
 * @note You can get a pointer to the new metaclass by calling \c object_getClass(newClass).
 * @note To create a new class, start by calling \c objc_allocateClassPair. 
 *  Then set the class's attributes with functions like \c class_addMethod and \c class_addIvar.
 *  When you are done building the class, call \c objc_registerClassPair. The new class is now ready for use.
 * @note Instance methods and instance variables should be added to the class itself. 
 *  Class methods should be added to the metaclass.
 */
OBJC_EXPORT Class objc_allocateClassPair(Class superclass, const char *name, 
                                         size_t extraBytes) 
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0);

```
##   **objc_registerClassPair(Class cls)**

注册这个类
```
/** 
 * Registers a class that was allocated using \c objc_allocateClassPair.
 * 
 * @param cls The class you want to register.
 */
OBJC_EXPORT void objc_registerClassPair(Class cls) 
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0);
```
## **objc_disposeClassPair(Class cls)**
销毁这个类 
```
/** 
 * Destroy a class and its associated metaclass. 
 * 
 * @param cls The class to be destroyed. It must have been allocated with 
 *  \c objc_allocateClassPair
 * 
 * @warning Do not call if instances of this class or a subclass exist.
 */
OBJC_EXPORT void objc_disposeClassPair(Class cls) 
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0);
```

##  **class_addProperty**
给类增加一个属性 
给类增加一个属性 给类增加一个属性 给类增加一个属性 给类增加一个属性 。给类增加一个属性 
给类增加一个属性 。给类增加一个属性 。给类增加一个属性
```
/** 
 * Adds a property to a class.
 * 
 * @param cls The class to modify.
 * @param name The name of the property.
 * @param attributes An array of property attributes.
 * @param attributeCount The number of attributes in \e attributes.
 * 
 * @return \c YES if the property was added successfully, otherwise \c NO
 *  (for example, the class already has that property).
 */
OBJC_EXPORT BOOL class_addProperty(Class cls, const char *name, const objc_property_attribute_t *attributes, unsigned int attributeCount)
    OBJC_AVAILABLE(10.7, 4.3, 9.0, 1.0);
```
代码实例
```
    //定义属性的属性 
        objc_property_attribute_t tq1 ={"T","@\"NSStting\""}; 添加一个NSString类型的类属性
        objc_property_attribute_t t2 = {"C",""}; copy
        objc_property_attribute_t t3 = {"N",""};nonamatic
        
        objc_property_attribute_t t4 = {"V","_dasda"}; 类属性的名字 。
        objc_property_attribute_t atre[] = {tq1,t2,t3,t4};
        
        ／／给存在的类添加属性 给存在的类添加属性
        ／／ 给存在的类添加属性
        class_addProperty(animal, "dasda", atre, 4);
        
       unsigned int ccc = 0;
       //获取类的属性列表 和数量 
        objc_property_t *ddfgg = class_copyPropertyList(animal, &ccc);
        
        
        for(int i=0;i<ccc;i++)
        {

            //获取属性的名字 获取舒心的名字 。
         const char *strwww =   property_getName(ddfgg[i]);
            
            获取属性的属性 。获取属性的属性 获取属性的属性
         const char *sdasdas =  property_getAttributes(ddfgg[i]);
            
            printf("=======%s\n======%s\n",strwww,sdasdas);
        }
        
        
        
        NSLog(@"==属性的数量===%d",ccc);
        

```
## **class_replaceProperty**

替换某个属性 替换某个属性 替换某个属性 替换某个属性 替换某个属性 替换某个属性
```
/** 
 * Replace a property of a class. 
 * 
 * @param cls The class to modify.
 * @param name The name of the property.
 * @param attributes An array of property attributes.
 * @param attributeCount The number of attributes in \e attributes. 
 */
OBJC_EXPORT void class_replaceProperty(Class cls, const char *name, const objc_property_attribute_t *attributes, unsigned int attributeCount)
    OBJC_AVAILABLE(10.7, 4.3, 9.0, 1.0);
```
这个方法 说好替换类的属性其实就是修改类的属性的属性 比如从NSString 类型变成NSarray类型
********
## **Working with Methods**
method 的一些方法 。method的一些方法 。一些方能发 一些方法 一些防范一些方法 。一些方法 一些方法 一些方法一些方法 一些方法 一些方啊 一些烦

## **method_getName(Method m)**
获取方法的名字 。获取方法 的名字 获取方法的名字 获取方法的名字 获取方法的名字 获取方法的名字
```
/* Working with Methods */

/** 
 * Returns the name of a method.
 * 
 * @param m The method to inspect.
 * 
 * @return A pointer of type SEL.
 * 
 * @note To get the method name as a C string, call \c sel_getName(method_getName(method)).
 */
OBJC_EXPORT SEL method_getName(Method m) 
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0);
```
## **method_getImplementation(Method m)**
获取是想方法的函数指针 获取是想方法的函数指针 获取实现方法的函数指针 获取实现方法的函数指针
获取实现方法的函数指针
```
/** 
 * Returns the implementation of a method.
 * 
 * @param m The method to inspect.
 * 
 * @return A function pointer of type IMP.
 */
OBJC_EXPORT IMP method_getImplementation(Method m) 
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0);
```
## ***method_getTypeEncoding(Method m)*
获取方法的返回值和参数的类型的信息 获取方法的返回值和参数的类型的信息
```
/** 
 * Returns a string describing a method's parameter and return types.
 * 
 * @param m The method to inspect.
 * 
 * @return A C string. The string may be \c NULL.
 */
OBJC_EXPORT const char *method_getTypeEncoding(Method m) 
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0);
```
## ****
获取方法的参数的个书 获取方法的参数的个数 获取方法的参数的个数
```
/** 
 * Returns the number of arguments accepted by a method.
 * 
 * @param m A pointer to a \c Method data structure. Pass the method in question.
 * 
 * @return An integer containing the number of arguments accepted by the given method.
 */
OBJC_EXPORT unsigned int method_getNumberOfArguments(Method m)
    OBJC_AVAILABLE(10.0, 2.0, 9.0, 1.0);
```

## **method_copyReturnType(Method m)**

## **method_copyArgumentType(Method m, unsigned int index)**
不过说很简单的一个方法
```
/** 
 * Returns a string describing a method's return type.
 * 
 * @param m The method to inspect.
 * 
 * @return A C string describing the return type. You must free the string with \c free().
 */
OBJC_EXPORT char *method_copyReturnType(Method m) 




    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0);
/** 
 * Returns a string describing a single parameter type of a method.
 * 
 * @param m The method to inspect.
 * @param index The index of the parameter to inspect.
 * 
 * @return A C string describing the type of the parameter at index \e index, or \c NULL
 *  if method has no parameter index \e index. You must free the string with \c free().
 */
OBJC_EXPORT char *method_copyArgumentType(Method m, unsigned int index) 
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0);

```

## **method_setImplementation(Method m, IMP imp) **
给方法赋值函数指针
给方法赋值函数指针 
给方法赋值函数指针 
给方法赋值函数指针
```
/** 
 * Sets the implementation of a method.
 * 
 * @param m The method for which to set an implementation.
 * @param imp The implemention to set to this method.
 * 
 * @return The previous implementation of the method.
 */
OBJC_EXPORT IMP method_setImplementation(Method m, IMP imp) 
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0);
```
## **method_exchangeImplementations(Method m1, Method m2)**
两个方法 更换函数指针 两个方法跟换函数指针 
两个方法跟换函数指针

/** 
 * Exchanges the implementations of two methods.
 * 
 * @param m1 Method to exchange with second method.
 * @param m2 Method to exchange with first method.
 * 
 * @note This is an atomic version of the following:
 *  \code 
 *  IMP imp1 = method_getImplementation(m1);
 *  IMP imp2 = method_getImplementation(m2);
 *  method_setImplementation(m1, imp2);
 *  method_setImplementation(m2, imp1);
 *  \endcode
 */
OBJC_EXPORT void method_exchangeImplementations(Method m1, Method m2) 
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0);
******
# *Working with Instance Variables* 

## **ivar_getName(Ivar v)**

获取属性的名称
```
/** 
 * Returns the name of an instance variable.
 * 
 * @param v The instance variable you want to enquire about.
 * 
 * @return A C string containing the instance variable's name.
 */
OBJC_EXPORT const char *ivar_getName(Ivar v) 
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0);
```
# *Working with Libraries*
## **objc_copyImageNames**
获取已经加载的静态库和胴体啊库
```
/** 
 * Returns the names of all the loaded Objective-C frameworks and dynamic
 * libraries.
 * 
 * @param outCount The number of names returned.
 * 
 * @return An array of C strings of names. Must be free()'d by caller.
 */
OBJC_EXPORT const char **objc_copyImageNames(unsigned int *outCount) 
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0);
```
## **class_getImageName**
```
Returns the dynamic library name a class originated from.
/** 
 * Returns the dynamic library name a class originated from.
 * 
 * @param cls The class you are inquiring about.
 * 
 * @return The name of the library containing this class.
 */
OBJC_EXPORT const char *class_getImageName(Class cls) 
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0);
```
## **返回一个库的所有的class**
```
/**
 * Returns the names of all the classes within a library.
 * @param image The library or framework you are inquiring about.
 * @param outCount The number of class names returned.
 * 
 * @return An array of C strings representing the class names.
 */
OBJC_EXPORT const char **objc_copyClassNamesForImage(const char *image, 
                                                     unsigned int *outCount) 
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0);
```

****
# *Working with Selectors*
**非常简单的几个sel的方法**

/** 
 * Returns the name of the method specified by a given selector.
 * 
 * @param sel A pointer of type \c SEL. Pass the selector whose name you wish to determine.
 * 
 * @return A C string indicating the name of the selector.
 */
OBJC_EXPORT const char *sel_getName(SEL sel)
    OBJC_AVAILABLE(10.0, 2.0, 9.0, 1.0);

```
/** 
 * Registers a method with the Objective-C runtime system, maps the method 
 * name to a selector, and returns the selector value.
 * 
 * @param str A pointer to a C string. Pass the name of the method you wish to register.
 * 
 * @return A pointer of type SEL specifying the selector for the named method.
 * 
 * @note You must register a method name with the Objective-C runtime system to obtain the
 *  method’s selector before you can add the method to a class definition. If the method name
 *  has already been registered, this function simply returns the selector.
 */
OBJC_EXPORT SEL sel_registerName(const char *str)
    OBJC_AVAILABLE(10.0, 2.0, 9.0, 1.0);

/** 
 * Returns a Boolean value that indicates whether two selectors are equal.
 * 
 * @param lhs The selector to compare with rhs.
 * @param rhs The selector to compare with lhs.
 * 
 * @return \c YES if \e lhs and \e rhs are equal, otherwise \c NO.
 * 
 * @note sel_isEqual is equivalent to ==.
 */
OBJC_EXPORT BOOL sel_isEqual(SEL lhs, SEL rhs) 
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0);

```

*****
# 关于函数指针的几个函数
```
/** 
 * Creates a pointer to a function that will call the block
 * when the method is called.
 * 
 * @param block The block that implements this method. Its signature should
 *  be: method_return_type ^(id self, method_args...). 
 *  The selector is not available as a parameter to this block.
 *  The block is copied with \c Block_copy().
 * 
 * @return The IMP that calls this block. Must be disposed of with
 *  \c imp_removeBlock.
 */
OBJC_EXPORT IMP imp_implementationWithBlock(id block)
    OBJC_AVAILABLE(10.7, 4.3, 9.0, 1.0);

/** 
 * Return the block associated with an IMP that was created using
 * \c imp_implementationWithBlock.
 * 
 * @param anImp The IMP that calls this block.
 * 
 * @return The block called by \e anImp.
 */
OBJC_EXPORT id imp_getBlock(IMP anImp)
    OBJC_AVAILABLE(10.7, 4.3, 9.0, 1.0);

/** 
 * Disassociates a block from an IMP that was created using
 * \c imp_implementationWithBlock and releases the copy of the 
 * block that was created.
 * 
 * @param anImp An IMP that was created using \c imp_implementationWithBlock.
 * 
 * @return YES if the block was released successfully, NO otherwise. 
 *  (For example, the block might not have been used to create an IMP previously).
 */
OBJC_EXPORT BOOL imp_removeBlock(IMP anImp)
    OBJC_AVAILABLE(10.7, 4.3, 9.0, 1.0);

/** 

```
## *objc_loadWeak(id *location)**
和 __weak的使用方法一样
```
/** 
 * This loads the object referenced by a weak pointer and returns it, after
 * retaining and autoreleasing the object to ensure that it stays alive
 * long enough for the caller to use it. This function would be used
 * anywhere a __weak variable is used in an expression.
 * 
 * @param location The weak pointer address
 * 
 * @return The object pointed to by \e location, or \c nil if \e location is \c nil.
 */
OBJC_EXPORT id objc_loadWeak(id *location)
    OBJC_AVAILABLE(10.7, 5.0, 9.0, 1.0);
```
##  **objc_storeWeak(id *location, id obj)**
不知道是干什么用的
```
/** 
 * This function stores a new value into a __weak variable. It would
 * be used anywhere a __weak variable is the target of an assignment.
 * 
 * @param location The address of the weak pointer itself
 * @param obj The new object this weak ptr should now point to
 * 
 * @return The value stored into \e location, i.e. \e obj
 */
OBJC_EXPORT id objc_storeWeak(id *location, id obj) 
    OBJC_AVAILABLE(10.7, 5.0, 9.0, 1.0);

```
下面几个方法 是根分类添加属性有关的几个方法 现在还不知道是干什么的 。。。。。


```
/** 
 * Sets an associated value for a given object using a given key and association policy.
 * 
 * @param object The source object for the association.
 * @param key The key for the association.
 * @param value The value to associate with the key key for object. Pass nil to clear an existing association.
 * @param policy The policy for the association. For possible values, see “Associative Object Behaviors.”
 * 
 * @see objc_setAssociatedObject
 * @see objc_removeAssociatedObjects
 */
OBJC_EXPORT void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy)
    OBJC_AVAILABLE(10.6, 3.1, 9.0, 1.0);

/** 
 * Returns the value associated with a given object for a given key.
 * 
 * @param object The source object for the association.
 * @param key The key for the association.
 * 
 * @return The value associated with the key \e key for \e object.
 * 
 * @see objc_setAssociatedObject
 */
OBJC_EXPORT id objc_getAssociatedObject(id object, const void *key)
    OBJC_AVAILABLE(10.6, 3.1, 9.0, 1.0);

/** 
 * Removes all associations for a given object.
 * 
 * @param object An object that maintains associated objects.
 * 
 * @note The main purpose of this function is to make it easy to return an object 
 *  to a "pristine state”. You should not use this function for general removal of
 *  associations from objects, since it also removes associations that other clients
 *  may have added to the object. Typically you should use \c objc_setAssociatedObject 
 *  with a nil value to clear an association.
 * 
 * @see objc_setAssociatedObject
 * @see objc_getAssociatedObject
 */
OBJC_EXPORT void objc_removeAssociatedObjects(id object)
    OBJC_AVAILABLE(10.6, 3.1, 9.0, 1.0);
```