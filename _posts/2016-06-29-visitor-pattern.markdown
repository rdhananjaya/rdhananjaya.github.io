---
layout: post
title:  "Visitor Pattern: The Mechanism"
summary: "Let's try to understand visitor pattern by thinking about how it's actually work under the hood, when to use it, and pros and cons of the visitor pattern."
date:   2016-06-29 10:16:36 +0530
categories: design
comments: true
page_id: visitorpattern
---

This post is my attempt to understand visitor pattern as much as possible. Although I had read about visitor pattern when learning OOD patterns, when I started to work with [program-transformations](http://www.program-transformation.org/), I started to use it heavily to work with abstract syntax trees.

Topics discussed in this post are:

- Definitions of Visitor pattern.
- When to use Visitor.
- Implementation.
- How it's work under the hood.

### Definition
Visitor let us define a new operations without changing the *classes* of the elements on which it operates.

[Scott Mayes](http://www.artima.com/cppsource/top_cpp_aha_moments.html):

> Visitor Pattern has nothing to do with visitation. Rather, it's a way to design hierarchies so that new virtual-acting functions can be added without changing the hierarchies.

Disambiguate heirarchical visitor pattern
[http://c2.com/cgi/wiki?HierarchicalVisitorPattern](http://c2.com/cgi/wiki?HierarchicalVisitorPattern)

### When to use Visitor
Visitor pattern is a good match when we have a fairly stable object structure, and want to operate multiple algorithms over it. All of the types in this structure need to be inheriting from a abstraction (or interface) having *visit* method.

Visitor allow us to decouple operations from the data model, and help us minimize pollution in data models and help keep all our related operations in one place.

For example let's assume we have an heterogeneous abstract syntax tree (AST) and we need to perform type checking and code generation over it. With visitor pattern we create two seperate visitor classes, one to perform type checking called **TypeCheckingVisitor** and another for code generation called **CodeGeneratingVisitor**. This way, we enforce single responsibility principle, by dividing type checking and code generation responsibilities amoung two classes.

Note: When not to use it.

Visitor pattern extracts seperate algorithms to their own classes, it make it easy to add, modify, and debug operations that operate on complex object structure. But adding a new class to original object structure forces us to add the releven method to visitor interface, hence overriding in in all the visitor implementation classes.
{:.inline-notes}

### The Pattern
Lets assume that we have an object structure where ASTNode is the supper class of all of the objects. For this discussion we'll consider direct descendants of ```ASTNode```, ```ForLoopNode```, ```AssigmentNode```, and ```DeclarationNode```, where all of the derived classes override ```accept(ASTVisitor v)``` method. ```ASTVisitor``` is a interface or abstract class containing visit methods for each derived class that needs to be visited.

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

![AST Node structure](https://blog.dhananjaya.me/static/image/c0d1831f.png)

All the implementations of **ASTNode** need to have a one method that accept instance of **ASTVisitor** and call the method dedicated for current node with argument of **this**, or a instance of current class.

Visitor interface and concrete visitor implementations.
![ASTVisitor Interface and concrete visitor implementations](https://blog.dhananjaya.me/static/image/c66d3d58.png)

All of the visitors has to have a method to handle visitation to each node, such as **visitDeclaration(DeclarationNode node)** which handle visitation for DeclarationNode.

Side Note:
We can use method overloading and name all the visitXXX methods to ```visit``` and let overload resolution handle the selection of appropriate method to execute.

 CodeGeneratingVisitor     |
---------------------------|
visit(DeclarationNode node)|
visit(AssigmentNode node)  |
visit(ForLoopNode node)    |
{:.uml-class-representation}

On the one hand it makes code concise, but on the other it's little bit easier to read and comprehend when we use seperate names. And I personally prefer naming methods differently to improve readability.

### Implementation
Object structure:

```java
interface ASTNode {
	accept(ASTVisitor v);
}

class ForLoopNode implements ASTNode {
	@Override
	accept(ASTVisitor v) {
		v.visitForLoop(this);
	}
}

class AssigmentNode implements ASTNode {
	@Override
	accept(ASTVisitor v) {
		v.visitAssigment(this);
	}
}

class DeclarationNode implements ASTNode {
	@Override
	accept(ASTVisitor v) {
		v.visitDeclaration(this);
	}
}
```

Visitors:

```java
interface ASTVisitor<T> {
	T visitForloopNode(ForLoopNode node);
	T visitAssigmentNode(AssigmentNode node);
	T visitDeclarationNode(DeclarationNode node);
}

class CodeGenVisitor implements ASTVisitor<String> {
	@Override
	String visitForLoopNode(ForLoopNode node) {
		
	}
	
	@Override
	String visitAssigmentNode(AssigmentNode node) {
	
	}
	
	@Override
	String visitDeclarationNode(DeclarationNode node) {
	
	}
}

class TypeCheckVisitor implements ASTVisitor<void> {
	@Override
	void visitForLoopNode(ForLoopNode node) {
		
	}
	
	@Override
	void visitAssigmentNode(AssigmentNode node) {
	
	}
	
	@Override
	void visitDeclarationNode(DeclarationNode node) {
	
	}
}
```

Usage:

```java

public class VisitorTestDriver {
	public static void main(String [] args) {
		AST tree = getAST(args); // code to get AST, probably some parser.
		TypeCheckingVisitor tcVisitor = new TypeCheckingVisitor();
		tree.accept(tcVisitor);
		
		CodeGeneratingVisitor cgVisitor = new CodeGenerationVisitor();
		String code = tree.accept(cgVisitor);
		
		// do stuff with *code*
	}
}

```

### How visitor works under the hood / theory
Visitor pattern is essentially a clean way to emulate double dispatch in single dispatched languages such as C++ and Java. With visitor pattern we can avoid runtime type checking (instanceof operator) and handover dispatching of appropriate method, to type system. 

In single dispatch languages two factors determine which method will execute among many, name of the method and the type of object instance upon which the method is being called.

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

When we need concrete implementations of ```Figure``` and ```Printer``` to corretly invoke appropriate printer on appropriate figure in ```figure.printOn(printer)``` we can employee double dispatch so that the type system automatically select with implementations to invoke.

Concreate Printer implementations 

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

Concreate Figure implementations

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

When we execute ```figure.printOn(printer)``` where figure is instance of **Circle**, and printer is instance of **InkjetPrinter**, ```printOn``` implementation of Circle class will be excecuted (using single dispatch). To that ```printOn``` method an instance of **InkjetPrinter** will be passed. When the method executes ```printer.printCircle(this);``` line, this will goto the **PrintCircle** implementation of **InkjetPrinter** and execute **PrintCircle** implementation on **InkjetPrinter**. Efectively invoking methods from two desired implementations, hence double dispatch.

Note: Dynamic vs Static dispatching

Dynamic dispatch is selecting a method amoung multiple implementations of polymorphic methods to execute in runtime. Where as in static dispatching, which method to execute is selected at compile time only using information available at compile time to the compiler.
{:.inline-notes}
In dynamic dispatching, caller does not need to know the exact type of the object upon which the method is being called. 
{:.inline-notes}

Note: Multiple dispatch

Multiple dispatch is the dynamically selecting to execute a method/function based on runtime attribute (type) of more than one of it's arguments. Here the type of the object in whitch this method is called upon or this message is sent to is also considered as one of it's arguments.
{:.inline-notes}


***** TODO how C# implement dispatching 
C# can archive multiple dispatch without using visitor pattern.
By casting to *dynamic*.


#### References and further readings..

[Stackoverflow discussion about when to use the visitor pattern.](http://stackoverflow.com/questions/255214/when-should-i-use-the-visitor-design-pattern)

[Robet C. Martin's explanation of visitor pattern with a practicle example.](http://www.butunclebob.com/ArticleS.UncleBob.IuseVisitor)

[http://codebetter.com/jeremymiller/2007/10/31/be-not-afraid-of-the-visitor-the-big-bad-composite-or-their-little-friend-double-dispatch/](http://codebetter.com/jeremymiller/2007/10/31/be-not-afraid-of-the-visitor-the-big-bad-composite-or-their-little-friend-double-dispatch/)

[https://blogs.msdn.microsoft.com/shawnhar/2011/04/05/visitor-and-multiple-dispatch-via-c-dynamic/](https://blogs.msdn.microsoft.com/shawnhar/2011/04/05/visitor-and-multiple-dispatch-via-c-dynamic/)
[https://blogs.msdn.microsoft.com/ericlippert/2011/03/17/implementing-the-virtual-method-pattern-in-c-part-one/](https://blogs.msdn.microsoft.com/ericlippert/2011/03/17/implementing-the-virtual-method-pattern-in-c-part-one/)

[http://www.objectmentor.com/resources/articles/visitor](http://www.objectmentor.com/resources/articles/visitor)

[http://c2.com/cgi/wiki?MultipleDispatch](http://c2.com/cgi/wiki?MultipleDispatch)

[http://c2.com/cgi/wiki?DoubleDispatchExample](http://c2.com/cgi/wiki?DoubleDispatchExample)

[http://c2.com/cgi/wiki?VisitorPattern](http://c2.com/cgi/wiki?VisitorPattern)
