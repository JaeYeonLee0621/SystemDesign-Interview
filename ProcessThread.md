# Program

## Static Program
- A program that doesn't change while it's running

# Process

![1](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/6e0a9dee-bf9f-43d7-910e-756edaa8d051)

- An instance of a computer program that is being executed

## Process Memory

![3](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/502c823f-3ccb-411d-a97f-9e5f1ade7985)

### Text/Code
- a read-only area where the compiled program code resides

### Data
- Initialized Data Section : Containing global and static variables
- Uninitialized Data Section (BSS) : This includes global and static variables that are not initialized by the programmer. Initially, the operating system initializes them to zero.

### Heap
- Begining at the end of the BSS section and grows dynamically at runtime as more memory is needed for dynamically allocated variables 

### Stack
- for storing the function call stack, which includes local variables, return addresses, and other function-related information

## Process resource sharing

![5](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/06a84c35-a25c-40e7-8178-28756e94adc6)


## Context Switch

### Problem of Context Switching on CPU Cache
- Cache Pollution : the previous process are replaced by entries needed by the newly scheduled process
- Clear or flush certain CPU Cache entries when switching between processe

### Solutions
- Tagged Caches : Some systems use tagged caches where cache lines are tagged with the process identifier (PID
- Selective Flushing : Sensitive contexts
- Cache Partitioning : Where parts of the cache are dedicated to specific processes or cores

## PCB (Process Control Block)

![9](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/ae075504-defc-4673-9765-88abe0876f19)

# Thread

![2](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/696ca1a8-d0c9-44e0-8800-ad8501eff3ed)

- A component of a process, meaning that multiple threads can exist within the same process, sharing resources such as memory and file handles
- Each thread maintains its own independent sequence of execution.

## Thread Memory

![4](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/1698f604-a3ba-43c5-9319-80ae4eaac108)

## Hyper Threading

![6](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/84453aff-18be-4a31-8aed-b355f4144ec9)

- Allows a single physical processor core to behave like two logical processors to the operating system and the applications running on it

![7](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/1fd2901a-0bb7-4237-9710-df2c99c02b8e)

# CPU Processing Method

## Parallelism

- the practice of executing multiple operations or tasks simultaneously to speed up processing and increase efficiency

## Concurrency

![8](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/f12b74f3-daac-4c0c-a742-2d083eaf32f2)

## TCB (Thread Control Block)

![10](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/c8f60e80-834e-4612-ab80-8f2b232c0ea6)
