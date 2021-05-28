# Object Oriented Language 
1. These can reduce/limit the affect of changes in the code.
2. Allow separations of concerns e.g. Algorithms can be de-entangled with OS details.


## SOLID principles
### What are the usual problems seen in Design.

Conside the game of black jack represented using following classes in python

```
class Card:
    def __init__(self):
        self.cards = [ (rank, suit) for rank in range(1,14)
            for suit in '♠♡♢♣' ]
        random.shuffle(self.cards)
    def deal(self):
        return self.cards.pop()
    def points(self, card):
        rank, suit = card
        if rank == 1: return (1,11)
        elif 2 <= rank < 11: return (rank, rank)
        else: return (10,10)

class Shoe(Card):
    def __init__(self, n):
        super().__init__()
        self.shoe = []
        for _ in range(n): self.shoe.extend(self.cards)
        random.shuffle(self.shoe)
    def shuffle_burn(self, n=100):
        random.shuffle(self.shoe)
        self.shoe = self.shoe[n:]
    def deal(self):
        return self.shoe.pop()
```

We have some problems like 
* Mixed Resposibilities : The class *Card* has methond *points* which calculates the value in a certain way. If the Game is changed or a new game has to be supported the *points* computation will need to change. 
* Missing Responsibilities : For a Black Jack game where the total is computed is not known.
* Limited reuse pottential : Cannot reuse for some other game like poker.
* Not substitutable : Card class cannot be substitued with another class (let say called) Deck.
* Poor Constructor design : Classes are tightly coupled through the constructor.
* Haphazard interface : No iterators in Deck or Shoe.

## SOLID - Guidlines to avoid design problems
1. S : Single resposibility
1.1 Kind of a summation of all the below. 

2. O : Open/Closed
2.1 What features needs to be exposed. 
2.2 A good class is open to be extended may by using derived classes.
2.3 A good class avoids necessasities of tweaking.

3. L : Liskov Substitution
3.1 Objects of some base class S can be replaced with objects of any derived class of S.
3.2 Constraints derived class design.
3.3 Helps with polymorphism.

4. I : Interface segregation
Helps with test case design.

5. D : Dependency inversion.
5.1 A direct dependency on a concrete class needs to be inverted.
5.2 depend on abstract classes. 

### Some other OO principles
6. TRY : Don't repeat yourself (no duplicate code.)
7. GRASP : General resposibility assigment software principles
8. TDD : Test-driven development.

### Example of SOLID implmentation 
In the card implmentation earlier lets apply 

#### Interface segregation: What it says is no client should be forced to depend on methods it does not use. The card and shoe (Deck) classes had 3 unrelated sets of features.
1. Creating a deck
2. Shuffling a card.
3. Compuation of points for a card.

The 1 and 2 are not used by a player. This should be segregated. Which leads to 3 conceptual classes
1. Card : Rank and Suit - used by the player.
2. Deck : Collection of cards. Support for Suffle and deal. Can be extended to create a multideck or shoe. 
3. Points system specific to black jack.

```
class Card:
    def __init__(self, rank, suit):
        self.rank= rank   # rank is immutable.
        self.suit= suit   # suit is immutable.
        self._text= "{:2d}".format(self.rank)
    def __str__(self):
        return "{_text:>2s}{suit}".format_map(vars(self))

class BlackjackCard(Card):
    """Eager calculation of points"""
    def __init__(self, *args):
        super().__init__(*args)
        self._hard = self.rank if self.rank <= 10 else 10
        self._soft = 11 if self.rank == 1 else self.rank if self.rank <= 10 else 10
    def hard(self):
        return self._hard
    def soft(self):
        return self._soft
	
Similarly we can extend it to work with PokerClass(Card)
```

Now lets do the same with Deck and shoe classes. Deck has 52 cards, Shoe has n Decks 2 <= n <= 9

Shoe supports Burn (Remove n cards from the shoe) suggests that it has to be a new class no Deck itself by priciples of interface segregation. Deck can be created using builder design pattern.

### Liskov substitution priciple
Any subclass should be replace any objects of super class without affecting the program behaviour. It is also known as strong behavioural subtyping.

```
class Shuffler:
	@staticmethod
	def shuffle(deck):
		pass
	
# Shuffle Class using a random module	

class Shuffler:
	@staticmethod
	def shuffle(deck):
		random.shuffle(deck)
```
 
Any application can pick up any of the above shuffler and use it. This is called late binding i.e the algorithm is choosen at the runtime.

If there is incompatibility then 
1. rethink :- is the interface required?
2. Refactor :- Should the interface be pushed to superclass.

#### Liskov substitution applies only upwards to superclass to any sibling of the superclass in a complex heirarchy.

#### If the constructor of an derived class is different from that of the base class, it will not allow the derived class to be used in place of base class. A way to work around is to use default parameter in constructor.

#### There should be almost never be a use case to use isinstance (dynamic_cast). But this is not a perfect world so it will be needed when

1. Type assertions for validation.
2. When used in comparison and in arithmetic.

### Open Close priciple
Open for extention and closed for modification. It summarizes the Interface segregation priciple and Liskov substitution priciple.

#### Dont use numeral parameters in an interface. 
#### No ripples in a buf fixes. 
		1. Extend the buggy class to create subclass and depricate the buggy one. 
		
### Dependency inversion priciple
Two elements
1. High level modules should not depend on low level modules.
2. abstaction should not depend on details and details should not depend on abstaction.

for example 
```
struct AwsCloud {
    void uploadToS3Bucket(string filepath) { /* ... */ }	
};

struct FileUploader {
	FileUploader(AwsCloud &Cloud);
	void scheduleUpload(string filepath);
};
```

It shows that Fileuploader is dependent on AWS cloud struct...this is a problem when we decide to swith to a new cloud. Ideally FileUploader should not be dependent on any cloud service.

Lets say we apply LSP and introduce a base class like this below 

``` 
struct Cloud {
	void uploadToS3Bucket(string filepath) = 0;
};

struct AwsCloud {
    void uploadToS3Bucket(string filepath) override { /* ... */ }	
};

struct FileUploader {
	FileUploader(Cloud &Cloud);
	void scheduleUpload(string filepath);
};

```
This is again a problem as the Cloud uses the concrete dependency of uploadToS3Bucket. As another cloud may have a different interface.

It should be modified as 
``` 
struct Cloud {
	void upload(string filepath) = 0;
};

struct AwsCloud {
    void upload(string filepath) override { /* ... */ }	
};
```

### Single reposponsibility Principle
A class should have a single reposponsibility. Consider an example
```
const string Prefix = "user-";

struct UserManager {
	UserManager(Database &DB) : db(DB) {
		// Add the prefix
		// Push to DB
	}
	string getUserReport() { 
		// Format the name to add prefix
		// Get user report from DB
	}
};
```

Problems here are 
1. The formatting of user name is done within usermanager.
2. The handling of DB directly. 

To resolve the problems it better to have 3 classes
1. Database manager.
2. Username formatter.
3. UserManager
