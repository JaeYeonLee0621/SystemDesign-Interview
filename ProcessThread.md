- [👩‍💻완전히 정복하는 프로세스 vs 스레드 개념](https://inpa.tistory.com/entry/%F0%9F%91%A9%E2%80%8D%F0%9F%92%BB-%ED%94%84%EB%A1%9C%EC%84%B8%EC%8A%A4-%E2%9A%94%EF%B8%8F-%EC%93%B0%EB%A0%88%EB%93%9C-%EC%B0%A8%EC%9D%B4)
- [🤔 스레드를 많이 쓸수록 항상 성능이 좋아질까?](https://inpa.tistory.com/entry/%F0%9F%91%A9%E2%80%8D%F0%9F%92%BB-Is-more-threads-always-better)

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

<hr/>

# MultiThreading

![1](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/c4fd2729-6886-4d9b-b710-461d22c08044)

# Application

## CPU Bound Application

![2](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/8ca44025-d63a-4c0e-a147-464e383d83a0)

- A CPU-bound application is one where the speed of the CPU is the primary factor determining the overall speed of the program
- Optimal Thread Cound = Core Count + 1

## I/O Bound Application

![3](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/6ea8ce12-c8e4-4e40-9d0e-e204a2da90e7)

- Threads can be over twice or three times more than Core Count
