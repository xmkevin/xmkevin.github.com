---
layout: post
title: "Objective C ivars' definition"
description: ""
category: Foundations
tags: [objc]
excerpt: "When I first learned obj-c, I confused on how to declare an instance variable properly, because there are so many ways to do it. This article is trying to figure out the differences of defining an instance variable and find the right way to create an instance variable in the right place."
---
{% include JB/setup %}

When I first learned obj-c, I confused on how to declare an instance variable properly, because there are so many ways to do it. This article is trying to figure out the differences of defining an instance variable and find the right way to create an instance variable in the right place.

###Decalre ivars in Header

Let's see the following code firstly, we define a Person class, which has name and age.

{% highlight objc %}

#import <Foundation/Foundation.h>

@interface Person : NSObject
{
    int _age;
    NSString *_name;
}

-(id)initWithName:(NSString *)name andAge:(int)age;

-(void)introduceMyself;

@end

//Implementation

#import "Person.h"

@implementation Person

-(id)initWithName:(NSString *)name andAge:(int)age
{
    self = [super init];
    
    if(self)
    {
        _name = name;
        _age = age;
    }
    
    return self;
}

-(void)introduceMyself
{
    NSLog(@"My name is %@, I am %d", _name,_age);
}

@end

{% endhighlight %}

It looks great, but what if I need some ivars which are not belonged to Person, just for computing or something else, should I declare them in header ?

{% highlight objc %}

#import <Foundation/Foundation.h>

@interface Person : NSObject
{
    int _age;
    NSString *_name;

    int _introduceCount;
}

-(id)initWithName:(NSString *)name andAge:(int)age;

-(void)introduceMyself;

@end

//Implementation

#import "Person.h"

@implementation Person

-(id)initWithName:(NSString *)name andAge:(int)age
{
    self = [super init];
    
    if(self)
    {
        _name = name;
        _age = age;
        _introduceCount = 0;
    }
    
    return self;
}

-(void)introduceMyself
{
    _introduceCount++;
    NSLog(@"My name is %@, I am %d. I have introduced myself %d times.", _name,_age, _introduceCount);
}

@end

{% endhighlight %}

If I want to use the Person class, what I care is the introduceMyself method, I don't care what ivars it defines. Therefore, it seems it is not a good idea to define ivars in header file. The header file is the interface for the class, everyone who wants to invoke the class does not care about what non-public ivars it has. Yes, there are some other ways to define ivars.

###Declare ivars in Extension

Let's take a look at the following code too.

{% highlight objc %}

//Extension
@interface Person()
{
    NSString *_name;
    int _age;
    int _introduceCount;
}

@end

{% endhighlight %}

We define ivars in Extension. The ivars in Extension are private. Now the header file is cleaner. Extension is used to declare private ivars, properties and methods. Here is another question: why should we use Extension, can we define them in Implementation directly? The answer is yes.

###Declare ivars in Implementation

Declaring iVars inside the implementation is definately a new construct in objective C. You need to be using xcode4.2 and have the LLVM compiler selected in your build settings. The idea is to keep your header files cleaner(from stackoverflow). So now, we can declare ivars in Implementation drectly like this:

{% highlight objc %}

//Implementation
@implementation Person
{
    int _age;
    NSString *_name;
}

-(id)initWithName:(NSString *)name andAge:(int)age
{
    self = [super init];
    
    if( self)
    {
        _name = name;
        _age = age;
    }
    
    return self;
}

-(void)introduceMyself
{
    NSLog(@"My name is %@, I am %d", _name,_age);
}

@end

{% endhighlight %}

###Conclution

Although we can define ivars in Header, Extension and Implementation. To make the header cleaner, we should only declare public and protected ivars in header. If you are using xcode4.2 or above and select the LLVM compiler, you can declare private ivars and methods inside Implementation directly.









