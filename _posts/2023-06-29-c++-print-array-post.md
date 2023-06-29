## How to print a container in C++?

In this post I describe a few ways to print a container in C++. For sake of brevity when I mention container 
I mean C++ std::vector, std::deque, std::list, std::set etc .

1. First, we simply iterate over all items of the container
   ```
   //Method 1
template <typename T>
void printArray1(std::vector<T>& myArray) {
  if (myArray.empty()) return;
  bool first = true;
  for (auto x : myArray) {
     std::cout << (first ? "" : " , ")  << x ;
     first = false;
  }
  std::cout << std::endl;
}
   ```
