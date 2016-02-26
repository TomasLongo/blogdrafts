---
layout: post
title:  "Learning Go - Interfaces and Method Receivers"
date:   2016-01-18
categories: go learning
comments: true
tags: [go,learning,interface]
---

While programming my first  programm with Go, I wanted to create an abstraction layer between my domain logic and the actual data sources. In my day work I would use a Java interface to do that. Piece of cake! So why not using Go interfaces just like I would use them in Java? Go does a great job letting interfaces look and feel like pointer to base classes/interfaces, but I encountered a situation where they did not behave like I expected. 

This blog post is my attempt to get a better understanding of the underlying mechanics of interface as well as improving my knowledge about Go itself. 

### Pointer vs. Non-Pointer receiver
A struct is the building block of every Go programm and can  be compared to a struct in C/C++. As in C/C++ a struct is a container that groups data that belongs logically together. 

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

### Implications of the receiver mechanics

Besides the value/reference characteristic, receiver types also define so called [method set](https://golang.org/ref/spec#Method_sets). A method set of a receiver type consists of all methods associated with that type and that fact comes along with a rule that specifies which type can access which method set. This implies that

* using a non-pointer variable, you can only access those methods that have been declared on a non-pointer receiver
* using a pointer variable, you can access methods declared on a pointer and non-pointer receiver

This fact has no meaning, as long as we invoke our methods on concrete types. Go is smart enough to try both ways of invoking a method. Look at the following snippet 


{% highlight go %}
type Foo struct {}
func (f *Foo) Baz() {...}

func main() {
	foo := Foo{}
	foo.Baz() // OK, even though method was defined on a pointer receiver
	(&foo).Baz() // <-- This is what Go tries behind the curtain
}
{% endhighlight %}

### Asssigning values to interfaces
Method sets will have an effect if we start to implement polymorphism with the `interface` type. Look at the following program:

{% highlight go %}
type Repository interface {
	Owner() string
	Name() string
}

type GithubRepo struct {
	name string
	owner string
}

func (g GithubRepo) Name() string {
	return g.name
}

func (g GithubRepo) Owner() string {
	return g.owner
}

func (g *GithubRepo) Fetch (url string) {
	// magically fetch the repo from github
	// and populate the fields
	g.name = "Awesome Repo"
	g.owner = "John Doe"
} 

// Print information for arbitrary repos. 
// We dont care what we are reading from as
// long as it satisfies the Repository interface
func printRepoInfo(repo Repository) {
	fmt.Printf("Repo %s belongs to %s", repo.Name(), repo.Owner())
}


func main() {
	githubRepo := GithubRepo{}
	githubRepo.Fetch("https://github.com/tomaslongo/repo")

	printRepoInfo(githubRepo)	
}
{% endhighlight %}

This is our scenario: We want to be able to fetch some information from a remote repository and print it to the screen. We don't really care what repo we are reading from. 

Our little program provides an interface, `Repository`, which exposes methods that gets us the informatin we want. In order to display this informaion we pass an instance of `Repository` to the function `printRepoInfo`.

The question at this point is, how does all the stuff about method receivers, pointer vs non-pointer affect our little snippet?

Let`s first analyse the current version of the snippet. If we compile and run it, Go will happily spit out the information of the repo we were asking for. Why?

### The Meaning of Method Sets
Because we provided a method set that includes the methods delcared by the interface. We defined the methods `Name` and `Owner` on non-pointer receivers and passed `printRepoInfo` a non-pointer. So, Go was able to find the interface methods in the method set of the type we provided.

Now let's break the program! All we have to do is to provide a method set that does not containt the methods to satisfy the interface. We do this by changing the declaration of `Name` and `Owner`. Instead of using non pointer-receivers we use pointer receivers.

{% highlight go %}
func (g *GithubRepo) Name() string {
	return g.name
}

func (g *GithubRepo) Owner() string {
	return g.owner
}
{% endhighlight  %}

Tryin to install this program does now fail and we get the following error message

`src/scratches/interfacetests.go:40: cannot use githubRepo (type GithubRepo) as type Repository in argument to printRepoInfo: GithubRepo does not implement Repository (Name method has pointer receiver)`

Go says that our passed type does not implement the interface methods, and is right with that claim simply because we did not provide the right method set. *(Remember what we concluded about method sets above! It comes back to us at this very moment)*

This error got me a whole while to understand because I was thinking of the interface type to be like interfaces in Java. But interfaces in Go are not built  to be used like that. On the surface, interfaces let you implement the good old 'is-a' relation and they also feel like pointers to base classes if you think in that terms.

The misunderstanding is that interfaces are not pointers, but should be seen more like containers. By assigning a value to an interface variable, Go simply sets up the fields of that container. An interface is made up of the underlying value, it is assigned and, most importantly, a table to functions that is populated with the method set of the underlying value that satisfies the interface delcaration.

![Interfaces under the hood]({{site.url}}/images/interfaces.png){: .centered-image}

Read this excellent [Article](http://research.swtch.com/interfaces) from Russ Cox (the image above is stolen from that article. Just replace `Stringer` for `Repository`) for an in depth discussion of how interfaces work under the hood.

In order to fix the error we have two choices:

1. We adapt the invokation of `printRepoInfo` to take in a pointer, which would assign a pointer to parameter variable
2. We revert our change and declare the methods `Name` and `Owner` on non-pointer receivers

Either way, it burns down to the fact that we do not have to provide the right type, but the right method set.

### tl;dr

Interfaces are a means to abstract away concrete types so that our programs do not have to have static knowledge about them. Their look and feel is intended to be much like that from classic OOP languages like Java and Go does a good job hiding the internal differences from the developer. However, there are cases where we should be aware of the fact that interfaces in Go are not classical pointers.

If you want to dig a bit deeper, take a look at [this nice article](http://jordanorelli.com/post/32665860244/how-to-use-interfaces-in-go) from Jordan Orelli. It was a valuable resource when researching about the topic.

