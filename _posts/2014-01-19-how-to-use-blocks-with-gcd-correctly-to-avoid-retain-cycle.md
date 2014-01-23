---
layout: post
title: "How to use Blocks with GCD correctly to avoid retain cycle"
description: ""
category: Foundations
tags: [objc, translation]
excerpt: "This is an article about Block, GCD and retain cycle."
---
{% include JB/setup %}

This is an translation article from [Chinese](http://tanqisen.github.io/blog/2013/04/19/gcd-block-cycle-retain/). I am trying my best to translate it, but because of my English limitation, it maybe not be translated exactly, if you have any question, please leave a comment.

## Intro of Block

As an extension of C, Block is not a new technology, while it is the same technology as closure and lambda expression of other languages. We should notice that there is not a GC for Objective-C in iOS, we should manage the memory by ourselves, and memory management for block is the point where we usually make mistakes. Error memory management could cause retain cycle or crash. Block is like function pointer, but the most difference of Block and function pointer is : Block can access external variables which are not in the function scope. In other words, Block is a function with execution context.

You can think that Block contains two compoments

1. The code that Block executes, which is generated when being compiled.
1. A data structure contains the external variables that the Block needs when execution.

Another difference of Block and Function pointer is that Block can be managed by `autoreleasepool` like Objective C object (But Block is not equal to Obj-C object, we will explain it later).

## Basic syntax of Block

{% highlight objc %}

// declare a Block variable
long (^sum) (int, int) = nil;
// sum is a Block variable, the Block type has two int parameters and a long return type.

// Define a Block and assign it to sum
sum = ^ long(int a, int b)
{
	return a + b;
};
// Invoke the Block
long s = sum(1,2);

{% endhighlight %}

Define a function which returns a block:

{% highlight objc %}

- (long (^)(int,int)) sumBlock
{
	int base=100;
	return [[^long(int a, int b)
	{
		return base + a + b;

	} copy] autorelease];
}

{% endhighlight %}

Looks strange? For looking comfortable, we can typedef the Block

{% highlight objc %}

typedef long (^BlkSum)(int,int);

-(BlkSum) sumBlock
{
	int base = 100;
	BlkSum blk = ^ long(int a, int b)
	{
		return base + a + b;
	}
	return [[blk copy] autorelease];
}

{% endhighlight %}

##Memory locations of Block

There are 3 types of Block according to the Block's position in memory, which are NSGlobakBlock, NSStackBlock and NSMallocBlock.

* NSGlobakBlock : like function, in text area;
* NSStackBlock : In Stack memory, would be released when function returned;
* NSMallocBlock : In Heap memory;

{% highlight objc %}

BlkSum blk1 = ^long(int a,int b)
{
	return a + b;
};
NSLog(@"blk1 = %@", blk1); // blk1 = <__NSGlobalBlock__: 0x47d0>

int base = 100;
BlkSum blk2 = ^long(int a, int b)
{
	return base + a + b;
};
NSLog(@"blk2 = %@", blk2); // blk2 = <__NSStackBlock__: 0xbfffddf8>

BlkSum blk3 = [[blk2 copy] autorelease];
NSLog(@"blk3 = %@", blk3); // blk3 = <__NSMallocBlock__: 0x902fda0>

{% endhighlight %}

Why the type of blk1 is NSGlobalBlock but the type of blk2 is NSStackBlock? The difference between blk1 and blk2 is that blk1 doesn't use any variable outside of the Block, therefore, it does not need to capture local variables, which make it the same as Function. I guess that the compiler put blk1 in the text area by the memory address 0x47d0. The only difference between blk2 and blk1 is blk2 uses the local variable base, while *defining* (not executing ) blk2, base was copied to the stack to be used by the Block. Executing the following code, the result is 203 rather than 204.

{% highlight objc %}

int base = 100;
base += 100;
BlkSum sum = ^ long (int a, int b) {
return base + a + b;
};
base++;
printf("%ld",sum(1,2));

{% endhighlight %}

In the Block, base is readonly, if we want to change the value of base, we should define it by appending `__block` attribute : `__block int base = 100`;

{% highlight objc %}

__block int base = 100;
base += 100;
BlkSum sum = ^ long (int a, int b) {
base += 10;
return base + a + b;
};
base++;
printf("%ld\n",sum(1,2));
printf("%d\n",base);

{% endhighlight %}

The console will print 214, 211. When using `__block` to declare a variable, the runtime value will be used instead of definition value. On the above example, when runing `sum(1,2)`, base will get the value of base++, which is 201, and then execute Block `base+=10; base+a+b;`, the result is 214. When Block execution finished, base has become 211.

## Copy, retain, release a Block

Differences to NSObject's copy, retain and release:

* Block_copy equals to copy and Block_release equals to release;
* Retain, copy and release a Block won't change the retainCount, the retainCount remains 1;
* NSGlobalBlock: retain,copy and release are invalid;
* NSStackBlock: retain and release operation are invalid. We should notice that the Block memory will be recycled when the NSStackBlock returns even it has been retained. A common mistake is `[mutableArray addObject:stackBlock]`, when the function is poped out of the stack, the stackBlock is recycled. The correct way is copy the block to the Heap firstly: `[mutableArray addObject:[[stackBlock copy] autorelease]`. Copy the block to generate a new NSMallocBlock object.
* NSMallocBlock: supports retain and release. Although the retainCount remains 1, but the memory management will increase and reduce the count. Copying a NSMallocBlock won't create a new instance, but it increase a reference count, like retain;
* Avoid retain a Block.

## Different types of variable in Block

### Basic types

* Local variables are readonly in Block. Block copys the variable value in definition, therefore, even the variable value has been changed outside of the Block, the variable value in the Block would stay the same.
{% highlight objc %}

int base = 100;
BlkSum sum = ^ long (int a, int b) {
  // base++; compile error, readonly
  return base + a + b;
};
base = 0;
printf("%ld\n",sum(1,2)); // The output is 103 rather than 3

{% endhighlight %}

* +static+ variables and global variables. If we change the base variable to static or global on above example, it can be readed and written in Block. Because global and static variables have fixed memory locations, Block reads the new value from memory directly, but not copied constant value in definition.

{% highlight objc %}

static int base = 100;
BlkSum sum = ^ long (int a, int b) {
  base++;
  return base + a + b;
};
base = 0;
printf("%d\n", base);
printf("%ld\n",sum(1,2)); // The output is 4 instead of 104
printf("%d\n", base);

{% endhighlight %}

The outputs are `0 4 1`, which indicates changing base outside of Block affects the value in the Block and vice versa.

* Block variable, variables with `__block` are called Block variables. Basic types of Block variables are equal to static and global variables.

### Assuming block1 is used by block2, when block2 is copied to Heap, block1 is copied too. But if block1 is a parameter, it won't be copied.

{% highlight objc %}
void foo() {
  int base = 100;
  BlkSum blk = ^ long (int a, int b) {
    return  base + a + b;
  };
  NSLog(@"%@", blk); // <__NSStackBlock__: 0xbfffdb40>
  bar(blk);
}

void bar(BlkSum sum_blk) {
  NSLog(@"%@",sum_blk); // The same as above, won't copied as a parameter

  void (^blk) (BlkSum) = ^ (BlkSum sum) {
    NSLog(@"%@",sum);     // No matter blk is in Stack or Heap, Block wount' be copied as a parameter
    NSLog(@"%@",sum_blk); // When blk is copied to the Heap, sum_blk is copied too.
  };
  blk(sum_blk); // blk is on Stack

  blk = [[blk copy] autorelease];
  blk(sum_blk); // blk is on Heap
}
{% endhighlight %}

### Objc objects, not like basic types, Block do change the retain count.

Code first

{% highlight objc %}

@interface MyClass : NSObject {
    NSObject* _instanceObj;
}
@end

@implementation MyClass

NSObject* __globalObj = nil;

- (id) init {
    if (self = [super init]) {
        _instanceObj = [[NSObject alloc] init];
    }
    return self;
}

- (void) test {
    static NSObject* __staticObj = nil;
    __globalObj = [[NSObject alloc] init];
    __staticObj = [[NSObject alloc] init];

    NSObject* localObj = [[NSObject alloc] init];
    __block NSObject* blockObj = [[NSObject alloc] init];

    typedef void (^MyBlock)(void) ;
    MyBlock aBlock = ^{
        NSLog(@"%@", __globalObj);
        NSLog(@"%@", __staticObj);
        NSLog(@"%@", _instanceObj);
        NSLog(@"%@", localObj);
        NSLog(@"%@", blockObj);
    };
    aBlock = [[aBlock copy] autorelease];
    aBlock();

    NSLog(@"%d", [__globalObj retainCount]);
    NSLog(@"%d", [__staticObj retainCount]);
    NSLog(@"%d", [_instanceObj retainCount]);
    NSLog(@"%d", [localObj retainCount]);
    NSLog(@"%d", [blockObj retainCount]);
}

@end

int main(int argc, char *argv[]) 
{
    @autoreleasepool {
        MyClass* obj = [[[MyClass alloc] init] autorelease];
        [obj test];
        return 0;
    }
}

{% endhighlight %}

The result is `1 1 1 2 1`

`__globalObj` and `__staticObj` have fixed memory location, therefore they won't be retained when Block is copied.

`_instanceObj` is not retained directly when Block is copied. Instead, it retains self. As a result, we can read and write `_instanceObj` in Block.

`localObj` is retained and the retainCount is increased by default when Block is copied.

`blockObj` is not retained when Block is copied.

*For those who are not ObjC objects, such as GCD dispatch_queue_t, are not retained when Block is copied. We should be careful of this case.*



### Retain cycle

The root of retain cycle is that Block and obj retain each other, as a consequence, nobody can be released. Taking the following code for example:

{% highlight objc %}
ASIHTTPRequest *request = [ASIHTTPRequest requestWithURL:url];
[request setCompletionBlock:^{
  NSString* string = [request responseString];
}];
{% endhighlight %}

{% highlight objc %}
       +-----------+           +-----------+
       | request   |           |   Block   |
  ---> |           | --------> |           |
       | retain 2  | <-------- | retain 1  |
       |           |           |           |
       +-----------+           +-----------+
{% endhighlight %}

The solution is use weak reference to break retain cycle:

{% highlight objc %}
__block ASIHTTPRequest *request = [ASIHTTPRequest requestWithURL:url];
[request setCompletionBlock:^{
  NSString* string = [request responseString];
}];
{% endhighlight %}

{% highlight objc %}
      +-----------+           +-----------+
      | request   |           |   Block   |
 ---->|           | --------> |           |
      | retain 1  | < - - - - | retain 1  |
      |           |   weak    |           |
      +-----------+           +-----------+
{% endhighlight %}

When `request` is released, the retainCount is 0. When request is dealloced, the Block will be released too.

{% highlight objc %}
      +-----------+           +-----------+
      | request   |           |   Block   |
 --X->|           | ----X---> |           |
      | retain 0  | < - - - - | retain 0  |
      |           |   weak    |           |
      +-----------+           +-----------+
{% endhighlight %}

The same trap as above:

{% highlight objc %}
self.myBlock = ^ {
  [self doSomething];
};
{% endhighlight %}

Here, self and myBlock retain each other. The solution is:

{% highlight objc %}
__block MyClass* weakSelf = self;
self.myBlock = ^ {
  [weakSelf doSomething];
};
{% endhighlight %}

{% highlight objc %}
@property (nonatomic, retain) NSString* someVar;

self.myBlock = ^ {
  NSLog(@"%@", _someVer);
};
{% endhighlight %}

Although we don't retain self in the Block, we use the member variable `_someVer`. Using member variables in Block, the block retains self rather than member variables. The solution is as before:

{% highlight objc %}
@property (nonatomic, retain) NSString* someVar;

__block MyClass* weakSelf = self;
self.myBlock = ^ {
  NSLog(@"%@", weakSelf.someVer);
};
{% endhighlight %}

Not only retain cycle happens in two objects, it happens in many objects, which are more complicated. 

{% highlight objc %}
ClassA* objA = [[[ClassA alloc] init] autorelease];
  objA.myBlock = ^{
    [self doSomething];
  };
  self.objA = objA;
{% endhighlight %}

{% highlight objc %}
  +-----------+           +-----------+           +-----------+
  |   self    |           |   objA    |           |   Block   |
  |           | --------> |           | --------> |           |
  | retain 1  |           | retain 1  |           | retain 1  |
  |           |           |           |           |           |
  +-----------+           +-----------+           +-----------+
       ^                                                |
       |                                                |
       +------------------------------------------------+
{% endhighlight %}

The same as before, using `__block` to break retain cycle

{% highlight objc %}
ClassA* objA = [[[ClassA alloc] init] autorelease];

__block MyClass* weakSelf = self;
objA.myBlock = ^{
  [weakSelf doSomething];
};
self.objA = objA;
{% endhighlight %}

 *NOTICE: In MRC `__block` will not be retained while `__block` is retained in ARC. In ARC, we should use `__weak` or `__unsafe_unretained` for weak reference. `__weak` can only be used in iOS5 or later.*

### Objects which Block uses are released in advance.

Looking at the following example, Not only do `request` holds Block, the other object also holds Block.

{% highlight objc %}
      +-----------+           +-----------+
      | request   |           |   Block   |   objA
 ---->|           | --------> |           |<--------
      | retain 1  | < - - - - | retain 2  |
      |           |   weak    |           |
      +-----------+           +-----------+
{% endhighlight %}

At the moment, if `request` was released by its owner

{% highlight objc %}
      +-----------+           +-----------+
      | request   |           |   Block   |   objA
 --X->|           | --------> |           |<--------
      | retain 0  | < - - - - | retain 1  |
      |           |   weak    |           |
      +-----------+           +-----------+
{% endhighlight %}

The `request` has been released, but the Block is still hold by the objA. If trigger the block now, it will crash the entire app since the block access the released request. To avoid this circumstance, developers should pay attention to the lifecycle of Block.

Another mistake which develoeprs usually make is using the `__block` incorrectly. For instance

{% highlight objc %}
__block kkProducView* weakSelf = self;
dispatch_async(dispatch_get_main_queue(), ^{
  weakSelf.xx = xx;
});
{% endhighlight %}

When passing Block to a dispatch_async as a parameter, the system copys the block to Heap. If the Block reference instance variables, it will retain self too. Since the dispatch_async does not know when the self will be released, dispatch_async must retain self by itself to avoid self was released unespectedly. In this case, dispatch_async does not increase the retain count of self, before Block was executed, self maybe has been released, which may cause crash.

{% highlight objc %}
// MyClass.m
- (void) test {
  __block MyClass* weakSelf = self;
  double delayInSeconds = 10.0;
  dispatch_time_t popTime = dispatch_time(DISPATCH_TIME_NOW, (int64_t)(delayInSeconds * NSEC_PER_SEC));
  dispatch_after(popTime, dispatch_get_main_queue(), ^(void){
    NSLog(@"%@", weakSelf);
});

// other.m
MyClass* obj = [[[MyClass alloc] init] autorelease];
[obj test];
{% endhighlight %}

The `dispatch_after` simulate an async task, executed 10 seconds later. The `obj` has been released while the block is executing, which cause crash. The solution is removing the `__block`.


 




