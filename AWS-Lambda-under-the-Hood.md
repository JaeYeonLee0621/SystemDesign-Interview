+) [AWS Lambda under the Hood](https://www.infoq.com/articles/aws-lambda-under-the-hood/?utm_campaign=infoq_content&utm_source=infoq&utm_medium=feed&utm_term=Development)

# AWS Lambda
- On-demand systems, requiring `quick resource provisioning` and release to prevent wastage
- `Scaling` rapidly in response to demand and efficiently scaling down to minimize wastage represents the scale tanet
- `Security` is the top prioirty at AWS, assuring users a safe and secure execution environment to run and trust their code
- `Performance` aiming to provide minimal environment to run and trust their code

# Invocation Model

![AWS Lambda Invoation Model](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/7efc2e50-6e8a-460e-a0db-6e560a947f66)

## Synchronous invoke
- A request is sent, routed to the execution environment and code is executed, providing a synchronous reponse on the same timeline

## Asynchronous invoke
- queueing the request, followed by executin at a different timeline through poller systems

# Invoke Request Routing

![Invoke Request Routing](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/1e558b7f-1c28-452b-b776-385dcdc64996)

- `Worker` : serving as the execution environment or sandbox
- Challenging : in an on-demand computing system, where the availability of workers or execution environments may be `uncertain during an invoke`

## Placement

- To address this, a new system called `placement` is introduce to create execution environments or sandboxes on-demands

<br/>

- Challenging : inclusion of on-demand initialization before each invoke request

+) [Multiple Steps]
1. Creating the execution environment
2. Downloading customer code
3. Starting a runtime
4. Initializing the function

![Invoke Late Distributionn](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/b33325e3-d27c-4d34-8130-fc97f0e3b92b)

## A worker manager

1. Pre-existing sandbox upon frontend request : leading to smooth "warm invokes"
2. The sbsence of a sandbos : initiates a slower path involving placement to create a new one

+) [Pre-existing sandbox predictions]
1. Usage Pattern
2. Predictive Analytics
3. Proactive Warm-Up : such as during periods of expected high traffic or in preparation for scheduled events
4. Dynamic Scaling

![Lambda Architecture](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/3225c7f9-32d6-48e4-8acc-069ec88134a5)

- `Worker manager` : responsible for tracking sandboxes, posed challenges due to to its reliance on in-memory storage, leading to potential data loss in case of host failure
- `Assignment service` : a reliable distributed storage known as the jornal log ensuring regional consistency
- `Leader-Follower Architecture` : facilitate failovers, fault-tolerant

# Compute Fabric

- The `worker fleet` within Lambda's infrastructure is responsible for executing code
- `Fleet` : EC2 instance, known as workers, where execution environments are created
- `Capacity Manager` ensures optimal fleet size adjustments based on demand and monitors worker health, promptly replacing unhealthy instances

![Lambda Compute Infrastructure](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/58e23706-7532-4e41-936a-c121727aa1a4)

- `Firecracker` : a fast virtualization technology
- By encapsulating each execution environment in a micro VM, ensures strong data isolation, allowing diverse accounts to coexist on the same worker securely
- Notable reduction in the cost of creating new execution environments
- `VM Snapshot` : distributing snapshots between workers, ensuring fast VM resumption, and maintaining strong security measures
- `Copy-On-Read` ~= `Copy-On-Write`

+) [Copy-On-Write]
- The system allows multiple processes to share access to the same data without creating a seperate copy
- If any process attempts to modify the data, the system creates a seperate copy before allowing the modification to occur

# Snapshot Distribution

- Traditional download method : time-consuming, taking at least 10 seconds to compute
- Snapshots are split into `smaller chunks`, typically `512 kilobytes`, `allowing the minimal set of chunks required for VM resumption to be downloaded first`
- When a process assesses memory, it either retrieves data from the memory pages or, if not available, falls back to snapshot file
- This snapshots are `encrypted with different keys`, categorized as operating system, runtime and function chunks
- `Operating System and runtime chunks can be shared`, leaveraging convergent encryption to plaintext content results in `equal encrypted chunks`
- This comprehensive approach enhances `cache locality`, increases hits to locally distributed caches and reduces latency overhaed
- In the production system, the indirection layer is replaced with a `sparse file system`, `streamlining the request process and providing chunks on-demand at the file system level`

# Have we solved cold starts

- An optimization in operating system involves `read-ahead`, where, anticipating `sequential reads` in regular files, multiple pages are read when a single page is accessed
- mapped memory instead of a regular file, which allows random access, this method proves inefficient, leading to the download of the entire snapshot file for seemingly random page requests

![Have we solved cold starts](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/db009fa4-722e-4046-b2dd-eb118660fd02)

- An analysis of memory access among 100 VMs reveals a consistent access pattern
- This pattern, recorded in a page access log, is attached to every snapshot
- during snapshot resumption, the system possesses `prior knowledge of the required pages and their sequence`, significantly enhancing efficiency
