[TOC]
## Item 44: Factor parameter-independent code out of templates.

***Tidbit : Member functions of class templates are implicitly instantiated only when used.***

Code duplication is a direct dis-advantage of templated classes for example consider the class 

```cpp
template<typename T, std::size_t n> 
class SquareMatrix { 
public:
	...
	void invert(); // invert the matrix in place
};

SquareMatrix<double, 5> sm1;
...
sm1.invert(); // call SquareMatrix<double, 5>::invert
SquareMatrix<double, 10> sm2;
...
sm2.invert(); // call SquareMatrix<double, 10>::invert
```

Two copies of invert will be instantiated here. The functions won’t be identical, because one will work on 5 ×5 matrices and one will work on 10 ×10 matrices, but other than the constants 5 and 10, the two functions will be the same.

One way to solve this bloat is to move the inverse to a separte class like below 

```cpp
template<typename T> // size-independent base class for
class SquareMatrixBase { // square matrices
protected:
...
	void invert(std::size_t matrixSize); // invert matrix of the given size
};

template<typename T, std::size_t n>
class SquareMatrix: private SquareMatrixBase<T> { //Private as the relation is composition.
private:
	using SquareMatrixBase<T>::invert; // make base class version of invert visible in this class; 
public:
	void invert() { invert(n); } // make inline call to base class
};

};
```

As the SquareMatrixBase is templatized only on the type, all matrices holding a given type of object will share a single SquareMatrixBase class. They will thus share a single copy of that class’s version of invert. Provided the base class invert(n) is inline or else we end up with the same problem.

SquareMatrixBase::invert is intended only to be a way for derived classes to avoid code replication, so it’s protected instead of being public. The additional cost of calling it should be zero, because derived classes’ inverts call the base class version using inline functions.

However we have not addressed how data is passed to the base class, this is done by extending like below 

```cpp
template<typename T>
class SquareMatrixBase {
protected:
	SquareMatrixBase(std::size_t n, T *pMem) // store matrix size and a
				   : size(n), pData(pMem) {} // ptr to matrix values
	void setDataPtr(T *ptr) { pData = ptr; } // reassign pData
...
private:
	std::size_t size; // size of matrix
	T *pData; // pointer to matrix values
};
```

Trade offs 
1. The invert with sizes have higher possibility of optimizations like constant propogation (substituting the values of known constants in expressions) and folding general instructions into immediate instructions. on the other hand having invert in the base class helps reducing the size of executable. 
2. Having a base class also increases the size of the object and reduces encapsulations (as data is also available with base class).

***Tidbit: Some linkers will not bloat code for STL containers with binary equivalent template parameters like vector<int> and vector<long> or list<int*>, list<const int*>, list<SquareMatrix<long, 3>*> etc.***

Things to remember:

1. Templates generate multiple classes and multiple functions, so any template code not dependent on a template parameter causes bloat.
2. Bloat due to non-type template parameters can often be eliminated by replacing template parameters with function parameters or class data members.
3. Bloat due to type parameters can be reduced by sharing implementations for instantiation types with identical binary representations.



## Item 43: Know how to access names in templatized base classes.
Consider the below example 

```cpp 
class CompanyA {
public:
...
	void sendCleartext(const std::string& msg);
	void sendEncrypted(const std::string& msg);
...
};

... // classes for other companies
class MsgInfo { ... }; // class for holding information used to create a message
template<typename Company>
class MsgSender {
public:
	... // ctors, dtor, etc.
	void sendClear(const MsgInfo& info) {
		std::string msg;
		create msg from info;
		Company c;
		c.sendCleartext(msg);
	}

	void sendSecret(const MsgInfo& info) // similar to sendClear, except
	{ ... } // calls c.sendEncrypted
};


// A specialized derived class to do logging and send a message. 
template<typename Company>
class LoggingMsgSender: public MsgSender<Company> {
public:
	... // ctors, dtor, etc.
	void sendClearMsg(const MsgInfo& info) {
		// write "before sending" info to the log;
		sendClear(info); // call base class function; This code will not compile!
		// write "after sending" info to the log;
	}
	...
};
```

The statements "c.sendCleartext(msg);" in derived class compiles while "sendClear" in sendClearMsg does not compile. The reason being that base class templates may be specialized and that such specializations may not offer the same interface as the general template. As a result, it generally refuses to look in templatized base classes for inherited names. In some sense, when we cross from Object-Oriented C++ to Template C++, inheritance stops working. The below template specialization of MsgInfo makes it concrete 

```cpp 
template<> // a total specialization of MsgSender; the same as the general template, except sendClear is omitted
class MsgSender<CompanyZ> { 
public: 
	... 
	void sendSecret(const MsgInfo& info)
	{ ... }
};
```
Thus for above derived class LoggingMsgSender passing "CompanyZ" as template parameter will result in error. There are however 3 different ways to invoke "sendClear(info);" 

```cpp
// 1. Use this pointer 
	this->sendClear(info);
	
// 2. Using "using" directive
	using MsgSender<Company>::sendClear;
	sendClear(info);

// 3. Using direct base class method invocation.
	MsgSender<Company>::sendClear(info); // Least desirable as explicit qualification turns off the virtual binding behavior.
	
	
A problem scenario like if we use CompanyZ as template parameter later will be diagnozed as an error at a later stage of compilation.


## Item 42: Understand the two meanings of typename.

Consider the example 

```cpp
template<class T> class Widget; // uses “class”
template<typename T> class Widget; // uses “typename”
```

As far as compiler is concerned there is not difference in the above declarations. But generally people use class when the template parameter can only be user defined types and typename if the template parameter is generic. There are exception to these as shown below

### Dependent and non-dependent names in a template 
Names in a template that are dependent on a template parameter are called dependent names. When a dependent name is nested inside a class, I call it a nested dependent name. C::const_iterator in below example is a nested dependent name. In fact, it’s a nested dependent type name, i.e., a nested dependent name that refers to a type.

```cpp
template<typename C> // print 2nd element in
void print2nd(const C& container) // container;
{ // this is not valid C++!
	if (container.size() >= 2) {
		C::const_iterator iter(container.begin()); // get iterator to 1st element; nested dependent name type
		++iter; // move iter to 2nd element
		int value = *iter; // copy that element to an int
		std::cout << value; // print the int
	}
}
```

This becomes important in scenario like below 

```cpp
C::const_iterator * x;
```

Here we do not know if C::const_iterator is a static member and whole statement means multiplication or if it is just a pointer declaration until we resolve C. There is an ambiguity during parsing for such statements. The rule for resolving this is simple 

### By default do not consider nested dependent names as types.

Thus the above statement 

```cpp
C::const_iterator iter(container.begin());
```

The C::const_iterator is not a type unless specified and hence it is not a valid C++ statement. We need to modify it with 

```cpp
typename C::const_iterator iter(container.begin());
```

Thus anytime we refer to a nested dependent type name in a template, we must immediately precede it by the word typename. Consider this example 

```cpp
template<typename C> // typename allowed (as is “class”)
void f(const C& container, // typename not allowed
		typename C::iterator iter); // typename required
```

Since C is not a nested dependent type name keyword typename is not allowed. The exception to this rule is in the typename must not precede nested dependent type names in a list of base classes or as a base class identifier in a member initialization list. For example:

```cpp
template<typename T>
class Derived: public Base<T>::Nested { // base class list: typename not
public: // allowed
	explicit Derived(int x) : Base<T>::Nested(x) { // base class identifier in mem. init. list: typename not allowed
		typename Base<T>::Nested temp; // use of nested dependent type name not in a base class list or as a base class identifier in a mem. init. list: typename required
		...
	}
};
```

Things to remember:

1. When declaring template parameters, class and typename are interchangeable.
2. Use typename to identify nested dependent type names, except in base class lists or as a base class identifier in a member initialization list.


## Item 41: Understand implicit interfaces and compiletime polymorphism.
Few definitions. Consider a function like below 

```cpp
void doProcessing(Widget& w)
{
	if (w.size() > 10 && w != someNastyWidget) {
		Widget temp(w);
		temp.normalize();
		temp.swap(w);
	}
}
```

We know that w has two types of interfaces

1. Explicit interfaces: The interfaces which are seen in the header file and these are something we can call. The behaviour does not change in runtime.
2. Polymorphic interfaces: These are virtual interfaces which is enabled/selected during runtime.

In template metaprogramming the above two characteristics take back seat instead we worry about implicit interfaces and compile time polymorphism. Consider the same example as 

```cpp
template<typename T>
void doProcessing(T& w)
{
	if (w.size() > 10 && w != someNastyWidget) {
		T temp(w);
		temp.normalize();
		temp.swap(w);
	}
}
```
Now we worry about the w of type T to support 

1. Interfaces size, normalize and swap member functions. Copy constructor for creating temp and comparison for in-equality. The set of interfaces that are valid for compilation of templates are known as implicit interfaces.
2. Calls that require instantiation of templates are catagorized into compile-time polymorphism. This is shown in the below example.

```cpp
template <class T>
void custom_add (T a, T b) {
    cout << "Template result = " << a + b << endl;
}

int main () {
    int   p = 1;
    int   i = 2;
    float n = 10.1;
    float e = 11.2;

    custom_add<int>(p, i);    // type specifier <int> present
    custom_add(n, e);         // no type specifier here
    // custom_add(p, e);      // this call will cause compile-time error
    return 0;
}
```

Things to remember:

1. Both classes and templates support interfaces and polymorphism.
2. For classes, interfaces are explicit and centered on function signatures. Polymorphism occurs at runtime through virtual functions.
3. For template parameters, interfaces are implicit and based on valid expressions. Polymorphism occurs during compilation through template instantiation and function overloading resolution



## Item 40: Use multiple inheritance judiciously

There are different problems like 

### Ambiguity
It becomes possible to inherit same name (e.g. function, typedef) from two different base classes. Consider the below e.g.

```cpp 
class BorrowableItem { // something a library lets you borrow
public:
	void checkOut(); // check the item out from the library
	...
};

class ElectronicGadget {
private:
	bool checkOut() const; // perform self-test, return whether
	... // test succeeds
};

class MP3Player: // note MI here
	public BorrowableItem, // (some libraries loan MP3 players)
	public ElectronicGadget
{ ... }; // class definition is unimportant

MP3Player mp;
mp.checkOut(); // Ambigous, error
mp.BorrowableItem::checkOut(); // This is how it is done
```

*Tidbit : In the above example the checkOut in class ElectronicGadget is not accessible but still call mp.checkOut() errors out. This is because the compiler first identifies the best match for the call then it looks for the accessiblity of that function. Here compiler gets confused as there are more than one match.*

### Deadly MI diamond
Consider the example 

```cpp
class File { ... };
class InputFile: public File { ... };
class OutputFile: public File { ... };
class IOFile: public InputFile,
public OutputFile
{ ... };
```

This enables IOFile to have two paths to base class File. And subsequently there is a question whether there should be multiple copies of members of class File. C++ supports both options with default to have replicates. So in above case the members are replicated. But if we want no replicates we should use virtual inheritance like below

```cpp 
class File { ... };
class InputFile: virtual public File { ... };
class OutputFile: virtual public File { ... };
class IOFile: public InputFile, public OutputFile
{ ... };

```

If we are concerned about correctness, public inheritance should always be virtual. But this cannot be a general as 
1. the data member access is costly for a virtual public inheritance 
2. the size of class is higher without virtual inheritance.
3. The initialization of base class is complex. The derived class must take the responsbility there however deep the base classes go.

So use them judiciously and follow the below rules

1. Don't use virtual inheritance
2. Even if you use them, do have data members in base class to avoid initialization perils


Things to remember 
1. Multiple inheritance is more complex than single inheritance. It can lead to new ambiguity issues and to the need for virtual inheritance.
2. Virtual inheritance imposes costs in size, speed, and complexity of initialization and assignment. It’s most practical when virtual base classes have no data.
3. Multiple inheritance does have legitimate uses. One scenario involves combining public inheritance from an Interface class with private inheritance from a class that helps with implementation.

## Item 39: Use private inheritance judiciously
If a class inherits another class privately then

1. Members inherited from base class are private to derived class.
2. Private inheritance means is-implemented-in-terms-of. If D privately inherits from B, it means that D objects are implemented in terms of B objects,

Private inheritance means nothing during software design, only during software implementation. Thus when to chose composition or private inheritance? 
*The answer is simple use composition when possible and use private inheritance when you must.*

Consider a widget which needs a timer we have following implementation alternatives. 

```cpp
class Timer {
public:
	explicit Timer(int tickFrequency);
	virtual void onTick() const; // automatically called for each tick
...
};

class Widget: private Timer {
private:
	virtual void onTick() const; // look at Widget usage data, etc.
...
};
```

As we need to modify onTick, we need to inherit from Timer. But since widget is not is-a Timer public inheritance is not the best. But again if we have public widget onTick modifying the behavior then clients may think it is callable. Now we see that private inheritance is not necessary if we use the below approach

```cpp
class Widget {
private:
	class WidgetTimer: public Timer {
	public:
		virtual void onTick() const;
		...
	};

WidgetTimer timer;
...
};
```
This has two advantages 
1. No derived class of widget can modify onTick
2. Reduce compilation dependencies


*When to consider private inheritance*
Consider a scenario

```cpp

class Empty {}; // has no data, so objects should
// use no memory
class HoldsAnInt { // should need only space for an int
private:
	int x;
	Empty e; // should require no memory
};
```

The size of Empty is not 0 but 1. Typically the compilers inserts a char to satisfy the class to be non-zero size free standing objects. If it is part of class HoldsAnInt then the alignment requirements kick in and it may use 8 bytes depending upon the compilers.

This case is not applicable to private inheritance as the base class is not free standing now. Thus in below the size is 4 now. This is known as empty base optimization.

```cpp
class HoldsAnInt: private Empty {
private:
	int x;
};
```

* TIDBIT : There are STL functions that are empty like unary_function or binary_function. Typically classes inherit from these and due to empty base optimization these do not add to size.*

Thus private inheritance is justified when there are
1. two classes not related by is-a where one either needs access to the protected members of another or needs to redefine one or more of its virtual functions.

## Item 38: Model "has-a" or "is-implemented-in-terms-of" through composition
Composition is the relationship between types that arises when objects of one type contain objects of another type. The term composition has lots of synonyms. It's also known as layering, containment, aggregation, and embedding. Composition has a meaning, too. In implmentation there are two domains, real world like people, animals etc and implmenation specifics like mutex, buffers, search trees etc. Thus
1. Composition is "has-a" for the real world.
2. Composition "is-implemented-in-terms-of" for implementation domain.

Tidbit : std library set implementation has a space cost of 3 pointers per element. 

### is-a and is-implemented-in-terms-of differences

If you want to implementat a set with std::list so that we save space, one might think this is better 

```cpp
class MySet : public std::list<int> {
...
};
```

This is wrong as Myset cannot contain duplicates however the std::list can contain that. Thus is-a relationship is not idea here. A better way is to 

```cpp
class MySet {

	std::list<int> MyList;
};
```
## Item 37: Never redefine a function's inherited default parameter value
Consider a scenario
```cpp
// a class for geometric shapes
class Shape {
public:
    enum ShapeColor { Red, Green, Blue };
    // all shapes must offer a function to draw themselves
    virtual void draw(ShapeColor color = Red) const = 0;
};

class Rectangle: public Shape {
public:
    // notice the different default parameter value — bad!
    virtual void draw(ShapeColor color = Green) const {
        std::cout << color;
    }
};

class Circle: public Shape {
public:
    virtual void draw(ShapeColor color) const {
        std::cout << color;
    }
};

int main() {
    Shape *ps; // static type = Shape*
    Shape *pc = new Circle; // static type = Shape*
    Shape *pr = new Rectangle; // static type = Shape*

    pr->draw();   // calls Rectangle::draw(Shape::Red)!
}
```

The reason for above behavior : virtual functions are dynamically bound, but default parameter values are statically bound. Why? To improve runtime efficiency as now compiler has to dynamically determine the default value also at runtime.

On the otherhand now lets say we give all the shapes same default parameter, this results in code duplication and it means if changed in the base class we need to change everywhere. To work around this use non-virtual interface idiom. e.g.

```cpp
void draw(ShapeColor color = Red) const // now non-virtual
{
	doDraw(color); // calls a virtual
}
...
private:
	virtual void doDraw(ShapeColor color) const = 0;
```
Now draw must never be overriden and hence the default value is always RED.

## Item 36: Never redefine an inherited non-virtual function
Consider a scenario
```cpp
class B {
public:
	void mf();
	...
};

class D: public B 
{ 
public:
	void mf(); // hides B::mf;	
};

D x; // x is an object of type D

B *pB = &x; // get pointer to x
pB->mf(); // call mf through pointer, calls B::mf

D *pD = &x; // get pointer to x
pD->mf(); // call mf through pointer, calls D::mf
```

- The reason for above is non-virtual functions are statically bound while virtul functions are dynamically bound. This case is true for references also. Thus it is not recomended to re-define the non-virtual members of a class. 
- The other way to see this is if D redefines mf, the D needs mf and also needs B's mf then D does not have a 'is-a' relationship with B. Conversly if B is-a D then mf must be declared virtual. 
- The same reason must apply desctructors of polymorphic class.


## Item 35: Consider alternatives to virtual functions
The are alternatives to define virtual functions are discussed here.
- Template Method Pattern via the Non-Virtual Interface Idiom : This school of thought says virtula methods should be declared private. The private virtual methods should be called from a public non-vitual method. This pattern strictly doesn't require to have the virtual function private, it can be protected.
	- Benefits are to enforce do before and do after work for the virtual function. e.g. locking a mutex.
	
- The Strategy Pattern via Function Pointers : Function the "virtual function" as a function pointer to the constructor of the class.
	- Benefits are like we can have different function for different instances of the class.
	- These functions do not have access to non public members of the class. Better abstraction. 
	
- Functors.
- The "Classic" Strategy Pattern : The way we put away the function in a a different classes virtual function. This pattern provides more flexibility on how to control the virtual function or specify this function with more derived classes. 

## Item 34: Differentiate between inheritance of interface and inheritance of implementation

As as class designer you may want to inherit 
1. Sometimes only the function interface  (like in the case of pure virtual function)
2. Sometime the implementation also.


For Pure virtual function it is cumbersome and duplication is involved so people generally make it virtual to simulate a default behavior. This can lead to undesirable side effects if a new derived class forget to overrid the default behaviour. Such a case can be avoided with 1. Pure virtual function and default implementation function. This default function must not be a virtual function.

To have a seperate declaration and definition for the same function is not acceptable to several people. Thus to avoid this we can implement the pure virtual function and use it as the default for derived class calls like 

```cpp
class Airplane {
public:
	virtual void fly(const Airport& destination) = 0;
...
};

void Airplane::fly(const Airport& destination) // an implementation of
{ 	// a pure virtual function
	// default code for flying an airplane to
	// the given destination
}

class ModelA: public Airplane {
public:
	virtual void fly(const Airport& destination) { 
		Airplane::fly(destination); 
	}
...
};
```

In essence, fly has been broken into its two fundamental components. Its declaration specifies its interface (which derived classes must use), while its definition specifies its default behavior (which derived classes may use, but only if they explicitly request it). A nonvirtual member function specifies an invariant over specialization i.e which does not change over specialization. In other words 

1. The purpose of declaring a non-virtual function is to have derived classes inherit a function interface as well as a mandatory implementation.


In implmentatin there is an empirical rule of 80-20 which says 80% of the runtime is spend executing 20% of the code. 

Thus in class design do not make the mistake of 
1. Declaring all the members as non-virtual as it leaves no room for specialization.
2. Declaring everything as virtual, that say class has nothing invariant for specialization.


1. Inheritance of interface is different from inheritance of implementation. Under public inheritance, derived classes always inherit base class interfaces.
2. Pure virtual functions specify inheritance of interface only.
3. Simple (impure) virtual functions specify inheritance of interface plus inheritance of a default implementation.
4. Non-virtual functions specify inheritance of interface plus inheritance of a mandatory implementation

## Item 33: Avoid hiding inherited names

Rules for name resolution scope
1. Look in the local scope.
2. Look in the scope of the class.
3. Look into the scope of base class
4. Look into the global scope.

These rule can not be applied directly to inheritance and overloading. Consider a scenario
```cpp
class Base {
private:
	int x;
public:
	virtual void mf1() = 0;
	virtual void mf1(int);
	virtual void mf2();
	void mf3();
	void mf3(double);
};

// and 
class Derived: public Base {
public:
	virtual void mf1();
	void mf3();
	void mf4();
};

Derived d;
int x;
...
d.mf1();    // fine, calls Derived::mf1
d.mf1(x);   // error! Derived::mf1 hides Base::mf1
d.mf2();    // fine, calls Base::mf2
d.mf3();    // fine, calls Derived::mf3
d.mf3(x);   // error! Derived::mf3 hides Base::mf3
```

However this can be circumvented using "using" directive like 

```cpp
class Derived: public Base {
public:
	using Base::mf1; // make all things in Base named mf1 and mf3
	using Base::mf3; // visible (and public) in Derived's scope
	virtual void mf1();
	void mf3();
	void mf4();
};

d.mf1();   // still fine, still calls Derived::mf1
d.mf1(x);  // now okay, calls Base::mf1
d.mf2();   // still fine, still calls Base::mf2
d.mf3();   // fine, calls Derived::mf3
d.mf3(x);  // now okay, calls Base::mf3
```

This only works when the class is inherited as public. If the class is inherited private then "using" directive will not work as using will make all the symbols visible in derived class. But it cannot be the case with private inheritance. The trick there is to use call forwarding i.e. call base function in derived function.

1. Names in derived classes hide names in base classes. Under public inheritance, this is never desirable.
2. To make hidden names visible again, employ using declarations or forwarding functions.

## Item 32: Inheritance and Object-Oriented Design

Public inheritance means "is-a." Everything that applies to base classes must also apply to derived classes, because every derived class object is a base class object. The derivied object should either implement a runtime error or make it private member.

## Item 27: Minimize casting
Different types of casts

1. C-style casts
```cpp
(T) expression // cast expression to be of type T
```
2. Function-style casts use this syntax:
```cpp
T(expression) // cast expression to be of type T
```

The above are old styles of casting and essentially means the same. C++ provide new casts as 

1. const_cast<T>(expression) : const_cast is typically used to cast away the constness of objects.
2. dynamic_cast<T>(expression) : is primarily used to perform "safe downcasting," i.e., to determine whether an object is of a particular type in an inheritance hierarchy. It is the only cast that cannot be performed using the old-style syntax. It is also the only cast that may have a significant runtime cost.
3. reinterpret_cast<T>(expression) : is intended for low-level casts that yield implementation-dependent (i.e., unportable) results, e.g., casting a pointer to an int. Such casts should be rare outside low-level code.
4. static_cast<T>(expression) : can be used to force implicit conversions (e.g., non-const object to const object, int to double, etc.). It can also be used to perform the reverse of many such conversions (e.g., void* pointers to typed pointers, pointer-to-base to pointer-to-derived), though it cannot cast from const to non-const objects.

Usually old style casting is not used these days. Only one place it looks is in explicit constructor call as shown below 


```cpp
class Widget {
public:
	explicit Widget(int size);
	...
};

void doSomeWork(const Widget& w); 

doSomeWork(Widget(15)); // create Widget from int with function-style cast
doSomeWork(static_cast<Widget>(15)); // create Widget from int with C++-style cast
```

Type conversions of any kind (either explicit via casts or implicit by compilers) often lead to code that is executed at
runtime. For example, in this code fragment,

```cpp 
double d = static_cast<double>(x)/y; // divide x by y, but use floating point division

/* OR */

Derived d;
Base *pb = &d; // Offset is applied runtime to derive base pointer.

``` 

Note : Casting object addresses to char* pointers and then using pointer arithmetic on them almost always yields undefined behavior.


## Item 26 : Postpone variable definitions as long as possible
1. Delay the Variable desclaration until it is needed. 
2. Delay a variable declaration until it can be constructed with initialization arguments. This way we replace default construction + assigment with copy constructor. For example change 
```cpp
	std::string encrypted;
	encrypted = password
	
	// with
	
	std::string encrypted(password);
```

In case of loops do no try to define variables inside the loop. Instead define them outside and assign them values. This way we avoid several constructions and destructions.

## Item 25 Consider support for a non-throwing swap

## Item 24 Declare non-member functions when type conversions should apply to all parameters
Consider the example of Rational class 
```cpp
class Rational {
public:
Rational(int numerator = 0, // ctor is deliberately not explicit;
int denominator = 1); // allows implicit int-to-Rational
// conversions
int numerator() const; // accessors for numerator and
int denominator() const; // denominator — see Item 22

const Rational operator*(const Rational& rhs) const;

private:
...
};

// Now this can be used as 

Rational oneEighth(1, 8);
Rational oneHalf(1, 2);
Rational result = oneHalf * oneEighth; // fine
result = result * oneEighth; // fine

// The below two statements are same
result = oneHalf * 2; //fine
result = oneHalf.operator*(2); // Implicit conversion of 2 to Rational.

// The below two statements are same
result = 2 * oneHalf; // error!
result = operator*(2, oneHalf); //Error
```

For the second statement to compile the compiler looks for 
1. An operator* support in 2, but obviously that doesn't exists.
2. An non-member operator* function that has a integer and Rational as parameters in the same namespace or in global namespace.

This problem is solved with a function in global space like 

```cpp
const Rational operator*(const Rational& lhs, // now a non-member
const Rational& rhs) // function
{
return Rational(lhs.numerator() * rhs.numerator(),
lhs.denominator() * rhs.denominator());
}
```

Should this be a friend function to the class : No as this can be implemented with public members of Rational it should not be friend to class Rational. Ideally declaring a function related to a class should not be declared friend function. 


## Item 23 Prefer non-member non-friend functions to member functions
Consider an example 

```cpp
class WebBrowser {
public:
...
void clearCache();
void clearHistory();
void removeCookies();
...

void clearEverything(); /* It just calls all the above methods */

...
};


// OR a non member function like below 
void clearBrowser(WebBrowser& wb)
{
wb.clearCache();
wb.clearHistory();
wb.removeCookies();
}
```

Which is better? 

Object-oriented principles dictate that data should be as encapsulated as possible. The greater something is encapsulated, then, the greater our ability to change. As a coarse-grained measure of how much code can see a piece of data, we can count the number of functions that can access that data: the more functions that can access it, the less encapsulated the data. 

This is also the reason that a member is public then unlimited number of functions can access it and it provides least encapsulation. Now considering our example providing member function clearEverything reduceds encapsulation compared to providing a non member function clearBrowser. This reasoning is based on two conditions
	1. The clearBrowser function is non-member and non-friend function.
	2. The function can be a non friend class's static member. Then it doesn't violate above principle.

In cpp the clearBrowser is defined in the same namespace.





## Item 22 Declare data members private
There are several reasons like 
1. Set the access rights to the variable appropriately. The variable can be read only, read-write or write only.
2. Encapsulation : The users need not know how the data is represented instead just how to access it. Encapsulatedness of a data member, is inversely proportional to the amount of code that might be broken if that data member changes.
3. protected is no more encapsulated than public.

## Item 21 Don't try to return a reference when you must return an object

Returning const reference is efficient but dont return a reference to a local object.

```cpp
const Rational& operator*(const Rational& lhs, // warning! bad code!
const Rational& rhs)
{
Rational result(lhs.n * rhs.n, lhs.d * rhs.d);
return result;
}
```

In the above scenario, it is obviously not a great idea to allocate with new since it doesn't provisions for free. Also it is not great to do return a static variable as it can be changed multiple times in an expression like 

```cpp
	if ((a * b) == (c * d)) { //Where all a, b, c, d are Rational objects.
```
The expression ((a*b) == (c*d)) will always evaluate to true, regardless of the values of a, b, c, and d!. This revelation is easiest to understand when the code is rewritten in its equivalent functional form:

```cpp
if (operator==(operator*(a, b), operator*(c, d)))
```

## Item 20 Prefer pass-by-reference-to-const to pass-by-value
1. Helps in avoiding object splicing. This is a situation when pass-by-value function accepts base class while it is passe

## Item 19 Treat class design as type design
Few questions to be answered before we create a class are 
1. How should objects of your new type be created and destroyed
2. How should object initialization differ from object assignment
3. What does it mean for objects of your new type to be passed by value
4. What are the restrictions on legal values for your new type
5. Does your new type fit into an inheritance graph
6. What kind of type conversions are allowed for your new type
7. What operators and functions make sense for the new type
8. What standard functions should be disallowed
9. Who should have access to the members of your new type
10. What is the "undeclared interface" of your new type
11. How general is your new type
12. Is a new type really what you need


## Item 18 Make interfaces easy to use correctly and hard to use incorrectly
1. Good interfaces are easy to use correctly and hard to use incorrectly. Your should strive for these characteristics in all your interfaces.
2. Ways to facilitate correct use include consistency in interfaces and behavioral compatibility with built-in types.
3. Ways to prevent errors include creating new types, restricting operations on types, constraining object values, and eliminating client resource management responsibilities.
4. const qualify the return values to avoid unexpected usage.


## Item 17 : Store newed objects in smart pointers in standalone statements

1. Store newed objects in smart pointers in standalone statements. Failure to do this can lead to subtle
resource leaks when exceptions are thrown.

One must be mindful of the execptions that can occur while creating the smart pointers. Consider a function and a calle like below

```
void Process(std::shared_ptr<Widget> pW, int priority)

// Call to the function is as 

Process(std::shared_ptr<Widget>(new Widget), getPriority());
```

The order of evaluation of the arguments is to the Process function is not defined by the c++ standard and can be determined by the compiler. So consider and order like this 
1. new Widget
2. getPriority()
3. Constructor for std::shared_ptr.

Now if getPriority call results in a execption the memory allocated by statement "new Widget" is lost. 

So it is good to have split into two statements like 

```
auto pW = std::shared_ptr<Widget>(new Widget);
Process(pW, getPriority());
```

## Item 16 : Use the same form in corresponding uses of new and delete.

1. If you use [] in a new expression, you must use [] in the corresponding delete expression. If you don't use [] in a new expression, you mustn't use [] in the corresponding delete
expression.
2. Avoid Typedef in Arrays.


During a new operator the object is created and its constructor is called. Similarly during the delete the destructor is called. Thus It is important to match the new and delete operators e.g consider

```
std::string *sPtr1 = new std::string;
std::string *sPtr2 = new std::string[100];

delete [] sPtr1; // Assume sPtr1 is an array and delete UB.
delete sPtr2;    // Only delete one element, rest in UB?

// Proper way to do this is 
delete sPtr1;
delete [] sPtr2;

```

Simply put if we use [] in new then [] must be used in delete. The problem arise when the object has different constructors which may end up using new with [] or not. Thus it becomes difficult in destructor to decide how to delete. Thus the rule is to use a form either with [] or not in all the constructors.

It is more critical in case of typedefs consider the following examples

```
typedef std::string[4] Address; 

MyAddress = new Address; // Equivalent to new string[4];

delete Address; // Undefined 
// need to use
delete [] Address;
```

Thus to avoid such cases abstain from typedefing arrays.

## Item 15 : Provide access to raw resource in resource managing class.

1. APIs often require access to raw resources, so each RAII class should offer a way to get at the resource it manages.
2. Access may be via explicit conversion or implicit conversion. In general, explicit conversion is safer, but implicit conversion is more convenient for clients.

There are generally two ways to accomplish this (Remember std::unique_ptr) viz explicit and implicit.
1. Explicit : provide a get() method.
2. Implicit : Provide overloading of * and -> operator.

One more way to do an implicit conversion is to overload the cast operator

```
BaseWrapper {

	operator() const { return *B; }

private:
	Base *B;
}

// This has a major downside considering the below assignments

BaseWrapper W1;
Base B1 = W1; // Implicit cast and copying underlying object.
```

This can result in cases when W1 is out of scope but B1 is still active.


## Item 14 : Think carefully in Resource managing classes

The copying of a RAII class may not always make sense so we have following cases 
1. Prevent copying : This is useful in case like below 

```
class lock_guard {
	lock_guard(mutex *mt) : m(mt) {
		lock(m);
	}
	
	~lock_guard() {
		unlock(m);
	}
private:
	mutex *m;
};

```

For the above 

``` 
mutex m1;
lock_guard g1(m1);
lock_guard g2(g1); // ??? What happens here
```

Doesn't makes sense. Thus delete the copy constructor. 

2. Reference counting underlying resource
Use std::shared_ptr on the resource.

3. Perform a deep copy. Sometimes copy is possible.

4. Transfer the ownership of the resource.

## Item 13 : Use objects to manage resources

Things to remember
1. To prevent resource leak use resource managing classes or RAII mechanism. It is also possbile to use reference-counting smart pointer (RCSP). Well these are much better understood in c++11 std::unique_ptr and std::shared_ptr context.

## Item 12 :  Copy all parts of an object. 
1. Copying functions should be sure to copy all of an object's data members and all of its base class parts.
2. Don't try to implement one of the copying functions in terms of the other. Instead, put common functionality in a third function that both call.

Usually the copy constructor and copy assignment should call respective functions in base class also here is an example.

```
class base {
public:
	std::string getName() { return Name; }
	
private:
	std::string Name;	
};

class Derived {
public:
	Derived(const Derived &Rhs) : Base(Rhs), Data(Rhs.Data) {}
	
	Derived &operator= (const Derived &Rhs) {
		Base.operator=(Rhs);
		Data = Rhs.Data;
	}
	
private:
	int Data;
};
```

## Item 11 : Handle assignment to self in operator= function.
Two key points 

1. Make sure operator= is well-behaved when an object is assigned to itself. Techniques include comparing addresses of source and target objects, careful statement ordering, and copy-and-swap.
2. Make sure that any function operating on more than one object behaves correctly if two or more of the objects are the same.

```
class widget {
	widget& operator=(widget &rhs) { 
		if (this == &rhs) return *this; // Return tacitly.
		
		return *this;
	}
};
```

## Item 10 : Have assignment operators return reference to *this

It helps with chain of assignment operation like 

```
x = y = z = 15; 		// x = (y = (z = 15));
```

The result of z = 15 is assigned to y. By returning the reference to \*this enables this kind of operation. Although it is not mandatory, any code otherwise also will compile. It is just the convention followed for built-in types and stdard library. This is also true for other assignment operators like +=, -= *= etc.

This also applies in cases when the rhs is of different type e.g.

```
class widget {
	widget& operator=(int rhs) { // Different type of rhs.
		...
		return *this;
	}
};
```

## Item 8: Prevent exceptions from leaving destructors.
Consider a scenario where 
```
class Widget {
public:
...
~Widget() { ... } // assume this might emit an exception
};

void doSomething()
{
	std::vector<Widget> v;
	...
}; 
```

## Item 7: Declare destructors virtual in polymorphic base classes
It is usual practive to allocate memory for derived class and use it after casting to a base class. Thus during the time of deletion it will invoke base class destructor instead of derived class destructor. Which can cause undefined behaviour. Thus declare the base class destructor virtual always. This problem can occur in a different way when using the STL classes like 
```
class SpecialString: public std::string { // bad idea! std::string has a non-virtual destructor
};
```

7.1 Usually a class will have virtual functions if it is intended for being used as a base class. If it is not declaring destructor virtual will make the size of class be increased. So the rule is use a virtual destructor only if you have a virtual function in your class. As a recommendation it is ok (?) to declare a pure virtual destructor if the base class is abstract class.

## Item 6: Explicitly disallow the use of compiler generated functions you do not want
I think this section is old. Just say "= delete" when not required. Or you can make then private in a base class.

## Item 5: Know what functions C++ silently writes and calls.
Usually when you declare and empty class, the compiler will declare a defualt constructor, destructor, copy constructor and copy assignment operator. If user has provided a constructor, compilers will not generate a default constructor. Thus 

```
class Empty{};

it’s essentially the same as if you’d written this:

class Empty {
public:
	Empty() { ... } // default constructor
	Empty(const Empty& rhs) { ... } // copy constructor
	~Empty() { ... } // destructor — see below
	// for whether it’s virtual
	Empty&operator=(const Empty& rhs) { ... } // copy assignment operator
};
```
As for the copy constructor and the copy assignment operator, the compiler-generated versions simply copy each non-static data member of the source object over to the target object. The built-in are copied bit by bit and the copy constructor is called for non-built-ins. 

Copy assignment is behaves almost similar to copy constructor but is bit more tricky as it can check for legality of copying the member variables e.g. 

```
template<typename T>
class NamedObject {
public:
	NamedObject(std::string& name, const T& value);
... 
	// assume no operator= is declared
private:
	std::string& nameValue; // this is now a reference
	const T objectValue; // this is now const
};
```
Now compiler is confused on how to initialize nameValue or objectValue as they are reference/const and can be initialized only once. So the confused compiler doesn't generates code for copy assignment. And if there is a assignment the compier puts out and error.

## Item 4: Make sure that objects are initialized before they’re used. Its better to initialize to avoid unwanted issues.
For non-member objects(?) of built-in types do it manually like
```
int x = 0;
const char *Text = "Name";
```
For everything else, the rule is simple. In all constructors initialize everything.
4.1 Assignments are diffferent than initializations. e.g.
```
class ABEntry {
	....
private:
	std::string theName;
	std::string theAddress;
	std::list<PhoneNumber> thePhones;
	int numTimesConsulted;	
};

ABEntry::ABEntry(const std::string& name, const std::string& address,
const std::list<PhoneNumber>& phones) {
	theName = name; 		// these are all assignments, not initializations
	theAddress = address; 
	thePhones = phones;
	numTimesConsulted = 0;
}
```

A better way to write the ABEntry constructor is to use the member initialization list instead of assignments:
```
ABEntry::ABEntry(const std::string& name, const std::string& address,
const std::list<PhoneNumber>& phones): 
    theName(name),
	theAddress(address), // these are now all initializations
	thePhones(phones),
	numTimesConsulted(0)
{}
```

This is called member initialization list. This way it is more efficient as the assignment above creates the variables "theName", "theAddress" etc using default constructor and then they are overwritten by the assignment. For built-in types the cost of assignment and member list initialization are same. However for consistency we can use the member list initialization. Similarly if the constructor doesn't take any parameters then also as a policy member list initialization can be performed. For example, if ABEntry had a constructor taking no parameters, it could be implemented like this:

```
ABEntry::ABEntry()
:   theName(), // call theName’s default ctor;
	theAddress(), // do the same for theAddress;
	thePhones(), // and for thePhones;
	numTimesConsulted(0) // but explicitly initialize it to 0.
{}
```

Member list initialization needs to be used in case of const reference or just const as they cannot be assigned.

4.2 The order of initialization in CPP classes is fixed to the order of declaration of the member variables. Thus in above class "theName" will always be initialized before "theAddress". This is true even if the order in member list initialization of these two are reversed. 

4.3 The initialization of static members in a class. The static object by definition exists from creation to end of program. There are local (in function/class) static variables and there are non-local static variables. The initialization order of non-local static objects is in-determinable. Thus we should not have a non-local static variable depend on another non-local static variable in another translation unit.

## Item 3: Use const whenever possible.
1. Use const iterators when possible.
2. Use Const member functions, member functions that differ in constness can be overloaded. e.g.
```
class TextBlock {
public:
...
	const char& operator[](std::size_t position) const // operator[] for
	{ return text[position]; } // const objects
	
	char& operator[](std::size_t position) // operator[] for
	{ return text[position]; } // non-const objects
	
private:
	std::string text;
};

void print(const TextBlock& ctb) // in this function, ctb is const
{
	std::cout << ctb[0]; // calls const TextBlock::operator[]
...
}

```

Two types of constness philosophy, bitwise (physical) and logical constness. Bitwise const says we can not change the contents of the class in a const member function. This is implemented by the compiler. e.g. 
```
class CTextBlock {
public:
...
	char& operator[](std::size_t position) const // inappropriate (but bitwise
	{ return pText[position]; } // const) declaration of
	// operator[]
private:
	char *pText;
};

const CTextBlock cctb("Hello"); // declare constant object
char *pc = &cctb[0];  // call the const operator[] to get a
					  // pointer to cctb’s data
*pc = ’J’; // cctb now has the value “Jello”
```

This type of implmentation is acceptable by the compiler but it isn't intuitive or logical.

1. Logical constness states that we can change the innerds of a class in a const function but the change should be oblivious to outside world. Thus comes handy *mutable* keyword.
2. Avoiding Duplication in const and Non-const Member Functions by calling the *const* function inside a *non-const* function with static and const casts.

## Item 2: Prefer consts, enums, and inlines to #defines.

Change something like 
```
#define ASPECT_RATIO 1.653
```

with 
```
const double AspectRatio = 1.653;
```
Advantages
1. Compiler can optimize and have better register usage.
2. Benefits from Static checks performed by the compiler.

### Class specific constants
To limit the scope of a constant to a class, declare it in the class. 

```
class GamePlayer {
private:
static const int NumTurns = 5; 	// constant declaration. It is not *definition*
int scores[NumTurns]; 			// use of constant
...
};
```
Ususally a definition is required for any variable declaration, but here it is an exception. This is true for integral types (char, bool, int). The compiler will allow it as long as we do not take the address of the variable. If we need the address then an additional statement like this is needed
```
const int GamePlayer::NumTurns; // in implmentation file. The definition of NumTurns, no value is given.
```

### Enum hacks
The values of an enumerated type can be used where ints are expected, so GamePlayer could just as well be defined like this:
```
class GamePlayer {
private:
enum { NumTurns = 5 }; // “the enum hack” — makes
// NumTurns a symbolic name for 5
int scores[NumTurns]; // fine
...
};
```
This technique takes advantage of the fact that the values of an enumerated type can be used where ints are expected.

1. For simple constants, prefer const objects or enums to #defines.
2. For function-like macros, prefer inline functions to #defines.

## Item 1 : CPP is a federation of language
Learning CPP can be broadly classified to following 4 areas
1. C 
2. Object oriented CPP
3. Template meta programming
4. and finally understanding STL

### STL
STL is a special library primarily consisting of containers, iterators, algorithms and function objects integrated beautifully. 

#### Container classes : Designed to hold and organize multiple instances of another type. 
  1. Generically containers can be classified into two based on the type of object they hold. *Value containers* which store the values and hence are responsible for creating and destroying owned values and *Reference containers* which hold reference or pointers to other objects. e.g. std::vector is an Value container and llvm::StringRef is a Reference container.
 
  1. STL Containers are classified into sequence, associative and container adaptors.
 
    * Sequence containers* maintain the ordering of the elements inserted and one can choose where to insert in a sequence container. e.g STL has 6 sequence containers std::vector, std::deque(pronounced “deck”), std::array, std::list, std::forward_list, and std::basic_string.
    * Associative container* : The elements are sorted when they are inserted. e.g. std::set, std::multiset (duplicates are allowed), std::map (associative array), std::multimap (disctionary, multiple values for keys are alowed)
    * Container adaptors : Predefined adaptors for specific uses. e.g. std::stack, std::queue, std::priority_queue

#### Iterators

Iterators are used to point at the memory addresses of STL containers. They are primarily used in sequence of numbers, characters etc. They reduce the complexity and execution time of program. There are 5 different types of iterators
1.  Input Iterators : Input iterators are considered to be the weakest as well as the simplest among all the iterators. These support
	1. One way traversal and hence can be used in single pass algorithms like std::find, std::equal, std::equal_range and std::count etc.
	2. Comparison check 
	3. Dereferencing 
	4. Incrementation (no decrements)
	
	Limitations
	1. No assignment supported.
	2. No decrement and arithmatic operations
	3. No relational operators support
	
	For e.g.
	```
	A == B     	// Allowed
	*A   		// Allowed
	A->m		// Allowed
	A++   		// Allowed
	++A   		// Allowed
	A--    		// Not allowed
	A + 1     	// Not allowed
	A <= B     	// Not Allowed
	```
2. Output Iterator : Just like input operators. Used for single pass algorithms. Only major difference is they can be written not read as in the examples below

	```
	*A = a 		// Allowed
	A++   		// Allowed
	++A   		// Allowed
	A == B     	// Not Allowed
	a = *A   	// Not Allowed
	A->m		// Not Allowed
	A--    		// Not allowed
	A + 1     	// Not allowed
	A <= B     	// Not Allowed
	```
3. Forward Iterator: Forward iterators are considered to be the combination of input as well as output iterators. They support all the operations except Decrement, relational and arithmatic operation for example 

	```
	*A = a 		// Allowed
	A++   		// Allowed
	++A   		// Allowed
	A == B     	// Allowed
	a = *A   	// Allowed
	A->m		// Allowed
	A--    		// Not allowed
	A + 1     	// Not allowed 
	A <= B     	// Not Allowed
	A[3]		// Not Allowed
	```
	
4. Bidirectional Iterators : Super set of Forward iterators but relational, derefence and arithmatic

	```
	A + 1     	// Not allowed 
	A <= B     	// Not Allowed
	A[3]		// Not Allowed
	```
	
5. Random-Access Iterators : Random-access iterators are iterators that can be used to access elements at an arbitrary offset position relative to the element they point to, offering the same functionality as pointers. They support all the above operations. Supported in vector, deque etc.
