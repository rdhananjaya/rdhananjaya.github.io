---
layout: post
title:  "Visitor Pattern: The Mechanism"
summary: ""
date:   2016-06-29 10:16:36 +0530
categories: design
comments: true
page_id: visitorpattern
---

Let's try to understand visitor pattern by thinking about how it's actually work under the hood.

- Definitions of Visitor pattern.
- When we can use it.
- Implementation.
- How it's work under the hood.

### Definition
Visitor let us define a new operation without changing the *classes* of the elements on which it operates.

[Scott Mayes](http://www.artima.com/cppsource/top_cpp_aha_moments.html):

> Visitor Pattern has nothing to do with visitation. Rather, it's a way to design hierarchies so that new virtual-acting functions can be added without changing the hierarchies.

Disambiguate heirarchical visitor pattern
[http://c2.com/cgi/wiki?HierarchicalVisitorPattern](http://c2.com/cgi/wiki?HierarchicalVisitorPattern)

### When to use Visitor
Visitor pattern is a good match when we have a fairly stable object structure, and want to operate multiple algorithms over it. All of the types in this structure need to be inheriting from a abstraction (or interface) having *visit* method.

Visitor allow us to decouple operations from the data model, and help us minimize pollution in data models and help keep all our related operations in one place.

For example let's assume we have an heterogeneous abstract syntax tree (AST) and we need to perform type checking and code generation over it. With visitor pattern we create two seperate visitor classes, one to perform type checking called **TypeCheckingVisitor** and another for code generation called **CodeGeneratingVisitor**. This way, we enforce single responsibility principle.

### The Pattern
**** TODO Write in English

[//]: # (***** Graph model 																															   )
[//]: # (http://yuml.me/diagram/scruffy/class/draw																									   )	
[//]: # ([ASTNode|accept(ASTVisitor v)]																												   )
[//]: # ([ASTNode]^-[DeclarationNode|accept(ASTVisitor v)]																							   )	
[//]: # ([ASTNode]^-[AssigmentNode|accept(ASTVisitor v)]																							   )	  
[//]: # ([ASTNode]^-[ForLoopNode|accept(ASTVisitor v)]																								   )	
[//]: # (																																			   )	 
[//]: # ([DeclarationNode]-accept method impl[v.visitDeclaration(this)]																				   )
[//]: # ([AssigmentNode]-accept method impl[v.visitAssigment(this)]																					   )
[//]: # ([ForLoopNode]-accept method impl[v.visitForLoop(this)]																						   )
[//]: # (																																			   )	 
[//]: # ([ASTVisitor|visitDeclaration(DeclarationNode node)|visitAssigment(AssigmentNode node)|visitForLoop(ForLoopNode node)]						   )	
[//]: # ([ASTVisitor]^-[TypeCheckVisitor|visitDeclaration(DeclarationNode node)|visitAssigment(AssigmentNode node)|visitForLoop(ForLoopNode node)]	   )	
[//]: # ([ASTVisitor]^-[CodeGenerationVisitor|visitDeclaration(DeclarationNode node)|visitAssigment(AssigmentNode node)|visitForLoop(ForLoopNode node)])

AST Node structure, this is the object structure we will be operating our algorithms on.

![AST Node structure](http://blog.dhananjaya.me/static/image/c0d1831f.png)

All the implementations of **ASTNode** need to have a one method that accept instance of **ASTVisitor** and call the method dedicated for current node with argument of **this**, or a instance of current class.

Visitor interface and concrete visitor implementations.
![ASTVisitor Interface and concrete visitor implementations](http://blog.dhananjaya.me/static/image/c66d3d58.png)

All of the visitors has to have a method to handle visitation to each node, such as **visitDeclaration(DeclarationNode node)** which handle visitation for DeclarationNode.

On the one hand we can use *visit* as the name of the method name and it will work just fine and be concise, but on the other it's little bit easier to read and comprehend when we use seperate names.

 CodeGeneratingVisitor     |
---------------------------|
visit(DeclarationNode node)|
visit(AssigmentNode node)  |
visit(ForLoopNode node)    |
{:.uml-class-representation}


### Implementation

### Disadvantages
Visitor pattern helps us seperate different algorithms to different classes, it make it easy to add, modify, and debug operations that operate on complex object structure. But adding a new class to that structure is more difficult when visitor pattern is involed.

### How visitor works under the hood / theory
Visitor pattern is essentially a clean way to emulate double dispatch in single dispatched languages such as C++ and Java. With visitor pattern we can avoid runtime type checking (instanceof operator) and handover dispatching of appropriate method, to type system. 

In single dispatch languages two factors determine which method will execute among many, name of the method and the type of object instance upon witch the method is being called.

In double dispatch which method to invoke is selected using name of the method, and two types of object instances.

For instance given two java interfaces

```java
 interface Figure {
	void printOn( Printer printer );
 }
 interface Printer {
	void printCircle( Circle circle );
	void printRectangle( Rectangle rectangle );
 }
```

And there implementations 

```java
 class InkjetPrinter implements Printer {
	public void printCircle( Circle circle ) {
	// ... rasterizing logic for inkjet printing of circles here ...
	System.out.println( "Inkjet printer prints a cirlce." );
	}
	public void printRectangle( Rectangle rectangle ) {
	// ... rasterizing logic for inkjet printing of rectangles here ...
	System.out.println( "Inkjet printer prints a rectangle." );
	}
 }
 class PostscriptPrinter implements Printer {
	public void printCircle( Circle circle ) {
	// ... postscript preprocessing logic for circles here ...
	System.out.println( "PostScript printer prints a cirlce." );
	}
	public void printRectangle( Rectangle rectangle ) {
	// ... postscript preprocessing logic for rectangles here ...
	System.out.println( "PostScript printer prints a rectangle." );
	}
 }
```

We need to create a figure.printOn(printer) method that will call appropriate *PrintFoo* method on approprite object instance, we achieve this using following code.

```java
 class Circle implements Figure {
	public void printOn( Printer printer ) {
        	printer.printCircle( this ); // <-- the "trick" !
	}
 }

 class Rectangle implements Figure {
	public void printOn( Printer printer ) {
	        printer.printRectangle( this );
	}
 }
```

Node how implementations of Figure interface simulate *double dispatch* by calling appropriate method with argument of *this* to select, Printer type and Figure type.




*** Dynamic vs Static dispatching.
Dynamic dispatch is selecting a method amoung multiple implementations of polymorphic methods to execute in runtime. Where as in static dispatching, which method to execute is selected at compile time only using information available at compile time to the compiler.

In dynamic dispatching, caller does not need to know the exact type of the object upon which the method is being called. 


**** single, double, multiple
Multiple dispatch is the dynamically selecting to execute a method/function based on runtime attribute (type) of more than one of it's arguments. Here the type of the object in whitch this method is called upon or this message is sent to is also considered as one of it's arguments. (Python's implecit self parameter).

For example in objectOfTypeA.method(x), both objectOftypeA and x are considered as arguments to method 'method'.

***** TODO how C# implement dispatching 
C# can archive multiple dispatch without using visitor pattern.
By casting to *dynamic*.
