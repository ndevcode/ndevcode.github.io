# How to print a container in C++?

In this post I describe a few ways to print a container in C++. When I mention
container I mean C++ std::vector, std::deque, std::list, std::set etc.

My objective is to pretty print the container. So if my input is 

```
          std::vector<int> myArray{1,2,3,4,5};
```

I would expect the output to be printed on screen as

```
          1 , 2 , 3 , 4 , 5 
```

without any extraneous commas either at the beginning or at the end of the line.


## Iterate over all items.

```
          //Method 1
          template <typename T>
          void printArray1(std::vector<T>& myArray) {
            if (myArray.empty()) return;
            bool firstTime = true;
            for (auto x : myArray) {
               std::cout << (firstTime ? "" : " , ")  << x ;
               firstTime = false;
            }
            std::cout << std::endl;
          }
```

One drawback with the above approach is that I need to make the check for 
'firstTime' in every iteration of the loop and print the comma. This is clumsy.

## A slightly improved approach.

```
          //Method 2
          template <typename T>
          void printArray2(std::vector<T>& myArray) {
            if (myArray.empty()) return;
            auto iter = std::begin(myArray);
            std::cout << *iter;
            std::advance(iter,1);
            for (; iter != myArray.end(); ++iter) {
              std::cout << " , " << *iter;
            }
            std::cout << std::endl;
          }
```

This approach requires me to print the first element of the array, manually 
advance the iterator to the next position and then check whether I've reached
the end of the container in the loop.

## Use the standard library algorithms to write to standard output.

The library function *std::copy* is useful in this context.

```
          //Method 3
          template <typename T>
          void printArray3(std::vector<T>& myArray) {
            if (myArray.empty()) return;
            std::copy(std::begin(myArray),std::prev(std::end(myArray)),std::ostream_iterator<T>(std::cout," , "));
            auto iter = std::prev(std::end(myArray));
            std::cout << *iter << std::endl;
          }
```

*std::ostream_iterator<T>(std::cout," , ")* is an iterator that writes elements
to the end of *std::cout*. The *,* is the optional delimiter that is printed 
after each element of the container is written. Note, that in this approach I 
iterate till the last but one element of the array and then print the last 
element standalone.

## Overloading the *<<* operator.

```
          //Method 4 
          template <typename T>
          std::ostream& operator<<(std::ostream& os, std::vector<T>& myArray) {
            if (myArray.empty()) return os;
            std::copy(std::begin(myArray),std::prev(std::end(myArray)),std::ostream_iterator<T>(os," , "));
            auto iter = std::prev(std::end(myArray));
            os << *iter;
            return os;
          }
```

This allows me to do print a container like below

```
          std::vector<int> myArray{1,2,3,4,5};
          std::cout << myArray << std::endl;
```

just as I would print an integer, double or std::string.

## A generic programming approach.

   The above approaches, while all valid, suffer from one serious drawback. They 
are all tied to the container type that is being printed. In the examples above 
I've implemented this code for std::vector. If I want to print std::list, 
std::deque or any of the other containers I will need to write additional code.

   Generic programming aims to solve this code bloat problem by making the code
independent of the container type used. As with the C++ standard library, I use 
iterators for my needs.

```
          //Method 5
          template <typename InputIt>
          void printArray5(InputIt left, InputIt right) {
            if (left == right) return ;
            auto iter = left;
            std::cout << *left;
            std::advance(iter,1);
            for (; iter != right; ++iter) {
              std::cout << " , " << *iter;
            }
            std::cout << std::endl;
          }
```

This allows me to do the following without code bloat

```
          std::vector<int> myArray{1,2,3,4,5};
          std::list<double> myArray2{1.1,2.2,3.3,4.4,5.5};
          printArray5(std::begin(myArray),std::end(myArray));
          printArray5(std::begin(myArray2),std::end(myArray2));
```
