[TOC]

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
