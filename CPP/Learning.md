# Effective CPP - Item 1
## CPP is a federation of language
Learning CPP can be broadly classified to following 4 areas
1. C 
2. Object oriented CPP
3. Template meta programming
4. and finally understanding STL

## STL
STL is a special library primarily consisting of containers, iterators, algorithms and function objects integrated beautifully. 
1. Container classes : Designed to hold and organize multiple instances of another type. 
  1. Generically containers can be classified into two based on the type of object they hold. *Value containers* which store the values and hence are responsible for creating and destroying owned values and *Reference containers* which hold reference or pointers to other objects. e.g. std::vector is an Value container and llvm::StringRef is a Reference container.
  1. STL Containers are classified into *sequence, associative and container adaptors*. 
    1. *Sequence containers* maintain the ordering of the elements inserted and one can choose where to insert in a sequence container. e.g STL has 6 sequence containers std::vector, std::deque(pronounced “deck”), std::array, std::list, std::forward_list, and std::basic_string.
    2. *Associative container : The elements are sorted when they are inserted. e.g. std::set, std::multiset (duplicates are allowed), std::map (associative array), std::multimap (disctionary, multiple values for keys are alowed)
    3. Container adaptors : Predefined adaptors for specific uses. e.g. std::stack, std::queue, std::priority_queue

### Iterator refresher

