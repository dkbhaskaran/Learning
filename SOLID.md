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

##### Builder design pattern
