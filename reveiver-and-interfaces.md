---
layout: post
title:  "About Interfaces and Method Receivers"
date:   2016-01-18
categories: go learning
comments: true
tags: [go,learning]
---

Article about the relation between interfaces and the receiver type of methods

[Intro]

Let's lay the foundation for this post and cover a few terms that we need to understand beforehand.

### Pointer vs. Non-Pointer receiver
A struct in Go can be compared to a struct in C/C++. As in C/C++ a struct is a data container that groups data that belongs logically together. 

But, unlike the C pendant, structs in Go can also be equiped with methods that can operate on a struct's data. This is done by declaring a method with a so called receiver. Consider following example:

{% highlight go %}
type Person struct {
	name string
	age int
}

func (p Person) MyNameIs() string {
	return p.name
}
{% endhighlight %}

`Person` holds a string and an int and has a method called `MyNameIs` which simply returns the name of the person. By prepending a type, we tell Go that the function should be invoked in the context of a value of type `Person`. This is our so called receiver.
 
Coming from the classic oop languages Java or C++ this syntax might seem a bit odd. But take a look at the following slight modification of the `MyNameIs` function:

{% highlight go %}
func (this Person) MyNameIs() string {
	return this.name
}
{% endhighlight %}

This should look more familiar right away. The receiver is the instance of the struct we're operating on.

Now, in the example above, we declared `MyNameIs` as attached to a non-pointer receiver, but we can also attach the function to a pointer receiver

{% highlight go %}
func (p *Person) MyNameIs() string {
	return p.name
}
{% endhighlight %}

Don't let the the syntax confuse you. `p` is an ordinary parameter that gets passed to the method which means that the same rules about passing parameters per value or reference apply to it:

* Declaring the receiver as a non-pointer means that, inside the method, we will be working on a copy of the struct's instance. E.g. modifications on the state of the passed instance will not reach the caller
* If we attach the method to a pointer, the caller will see the changes on the instance we made inside the method.

So when to use which syntax? Here is a simple rule of thumb:

> If you just have to read data from the struct, use a non-pointer receiver. Otherwise, go with a pointer.

#### Implications of the receiver mechanics

Besides the value/reference characteristic, receiver types also define so called [spec](method sets). A method set of a receiver type consists of all methods associated with that type and that fact comes along with a rule that specifies which type can access which method set. This implies that

* using a non-pointer variable, you can only access those methods that have been declared on a non-pointer receiver
* using a pointer variable, you can access methods declared on a pointer and non-pointer receiver

This fact has no meaning, as long as we invkoe our methods on concrete types. Go is smart enough to try both ways of invoking a method. Look at the following snippet 


{% highlight go %}
type Foo struct {}
func (f *Foo) Baz() {...}

func main() {
	foo := Foo{}
	foo.Baz() // OK, even though method was defined on a pointer receiver
	(&foo).Baz() // <-- This is was Go tries behind the curtain
}
{% endhighlight %}

#### Asssigning values to interfaces
Method sets will have an effect if we start to implement polymophism with the `interface` type. Interfaces are used to define a set of methods. Think of it as a protocol that describes how objects should communicte with each other.  


* pointer vs. non-pointer receiver
  * Link to article from guy as well as to go faq. Give a short tl;dr though
  * tl;dr 
    * Value receivers's method set contains methods declared with a method receiver
      * code
    * Pointer receivers contain methods from pointer receiver declarations as well as
      from value receivers decalrations
      * code
  * Can I call methods that where decalared with a pointer receiver from a value receiver??
    * This is possible: 
      [code]
    * Why is this possible?
      * Go is smart and will try to dereference the pointer of the value and call the method by itself
        [code]
        foo := Foo{}
        foo.pointerMethod() // --> OK
        (&foo).pointerMethod() // --> This is what go is doing behind the curtains

* how do pointer vs non-pointer receiver affect the passing of objects to interfaces??
* What is an interface in the 1st place? [mention cox article]
  * An interface is represented internally by two pointers:
    * An interface is a *type* of its own.
    * One points the underlying value, e.g. whatever is assigned to the interface variable
    * The other points to a table of pointers. The pointers in this table point to the underlying values methods that macht the declaration of the interface.
    * [code sample]
      	type Greeter interface {
   		MyNameIs() string
		SayHiTo(otherPerson Person) string
	}
	
	type Person struct {
		name string
	}

	func (j Person) MyNameIs() string {
		return j.name
	}

	func (j Person) SayHiTo(otherPerson Greeter) string {
		return "Hi " + otherPerson.MyNameIs() + " how are you?"
	}

	func main() {
		john := John{name: "John"}
		max  := Max{name: "Max}	
		
		john.SayHiTo(max)
	}
      [code sample end]
    
    * We are interested in function `SayHiTo`, which takes in a paramter of type `Person`. Now, how does `Person` look like under the hood?
      As we said earlier, interfaces are represented by two pointers. In this case:
        * Pointer one points to the type `Person` which we passed in the main function
        * Pointer two points the function `SayHiTo` which is implemenented by the specific instance of `Person`, `Max` in this case.	
      the Invokiation of `SayHiTo` on the passed `Greeter` object is now handled is now handled by the interface. 
        * Alternatices
          * a) since the interface holds the table of methods it does not need to dereference the underlying value to invoke the method
          * b) it dereferences the underlying value 
      Up to this point we should be fine. Compiling and running our little programm should make John say hi to max.

* Invoking a method that was declared on a pointer receiver on an interface that has an underlying non pointer value
  * Instead of declaring 'SayHiTo' for value, we now assign it to a pointer receiver 
    * [code sample]
	// same declarations as above
        ...

	func (j *Person) SayHiTo(oterPerson *Greeter) string {
		return "Hi " + otherPerson.MyNameIs() + " how are you?"
	}
      [code sample end]
    * What we now get is the following error message from Go
    * What's the matter here???

