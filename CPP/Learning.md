[TOC]

## Item 31: Minimize compilation dependencies between files.

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
