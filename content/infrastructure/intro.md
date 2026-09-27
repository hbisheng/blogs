---
title: "Basic Infrastructure Components"
date: "2023-05-28"
weight: 10
---

This post talks about basic infrastructure components that are regarded as common knowledge among software engineers.

## Infrastructure components

### **Code management**

Code is at the core of software engineering. **Git** can be used to manage the codebase; it provides version control and allows for large-scale collaborations.

Consider this everyday scenario: a software engineer needs to make some modifications to the existing code. The engineer first pulls the latest version of the code from Git and initiates a new feature branch for the intended changes. Upon committing these changes to the feature branch, a 'diff' (the set of differences between the new code and the existing code) is submitted for peer review. Once all comments are addressed and approvals obtained, the 'diff' is 'landed', i.e., merged into the main branch.

### **Tests**

Your changes should contain relevant **unit tests** and **integration tests**. Tests help ensure the functionality and compatibility of code changes and help prevent potential bugs.

- Unit tests: These test the smallest independent units of code, such as functions or classes. Dependencies are usually mocked (simulated) so that the behaviors can be controlled, as these dependencies are not the primary focus of testing.
- Integration tests: Test how different components work together. Instances of dependencies are launched before you run your core test logic. This is to avoid failures where components work fine independently but fail when connected.

To further enhance code reliability, automated testing infrastructure runs these tests before any code is merged onto the main branch, keeping it free of test failures. In cases where a commit slips into the main branch and breaks tests, it should be reverted.

### **Code build**

Before you can run your code in production, code needs to be built into binaries along with its dependencies. The output is typically a package or container image, which can be facilitated by build frameworks like Bazel. Once built, these packages are typically stored in blob storage systems such as Amazon S3, and later distributed to the target hosts.

### **Code deployment**

Code deployment in a microservice environment involves running your service code on a specific host. This is usually managed by an orchestration platform. Here, all you need to do is specify a package and define any resource requirements in your request. The platform then finds an available host and starts your service. The details of orchestration will be elaborated on in a subsequent post.

### **Service Discovery**

Services often interact with each other. But how does a service locate another? This is where service discovery comes in. Essentially, service discovery is a directory that maps services to their target endpoints, making this information accessible for queries. This directory could be stored anywhere—a good example is etcd.

Service discovery should also handle changes in endpoints, whether due to host failures or maintenance tasks. For single-instance services like MySQL, DNS could be used for discovery; however, this method might not be very scalable and could require hardcoded ports.

### **Code monitoring (Observability)**

- **Metrics**: By emitting metrics from your code and storing them in a time series database, you can monitor your code's behavior over time. These metrics can be displayed on platforms like Grafana, and be queried by alerts and humans.
- **Logging**: Process logs can provide valuable insights for troubleshooting bugs and understanding the sequence of events. Structured logging enables you to perform full-text searches and filter or group by tags. Loki, for instance, is a popular centralized logging system.
- **Tracing**: Tools like Jaeger can take a sample of requests and measure the time taken from beginning to end, providing insights into any performance bottlenecks.

### Health checking

Health checking is needed at different layers to promptly detect and handle failures in the system. The orchestration platform typically has health checking probes for the services. There may also be dedicated services that ping other services or run custom logic to verify the services' functionalities as a blackbox.

I stumbled across the question “Who performs health checking for the health checking service?”. The simple answer seems to be, just make the health checking service failure tolerant (e.g. running standby instances) and set up proper alerting.

### Other

Infrastructure in general has a lot of other components like networking and hardware management. But we won't get into those for now.

## Appendix

### Common Git commands

```
git status
git add .
git commit
git pull
git checkout <branch>
git rebase -i <branch>
git merge-base <branch-A> <branch-B>
git diff <branch>
```

### Bazel knowledge

Hermeticity: Every build in Bazel is isolated from the system. It has access only to the declared inputs and the specified build environment. As a result, reproducibility is achieved: the build result is the same across environments/systems.

Dependencies and integration testing are powerful.

Bazel specifies the layout of code files in the syntax of code.

### Loki Usage

|  |  |  |
| --- | --- | --- |
| **Operator** | **Meaning** | **Example** |
| |= | include a string | {app="my-app",env="dev"} |= "error" |
| |~ | matches a regex | {app="my-app"} |~ "error.\*" |
| != | exclude a string |  |
| !~ | exclude a regex |  |
| and | Combining operators (applying independently) | {app="my-app"} |= "error" and |= "timeout" |
|  | Multiple matchers (applying simultaneously) | {app="my-app"} |= "error" |~ ".\*timeout.\*" |
