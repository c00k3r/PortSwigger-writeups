# Race Conditions – Learning Path Overview

## Overview

This learning path explores race condition vulnerabilities in web applications, where the outcome of an operation depends on the timing or order in which multiple requests are processed.

Race conditions occur when an application handles concurrent requests in an unexpected way, potentially allowing attackers to bypass security controls, manipulate application state, or perform actions that should not normally be possible.

Throughout this learning path, I explored different types of race conditions, learned how to identify race windows, and practiced exploiting them using Burp Suite.

## Key Concepts Covered

### 1. Understanding Race Conditions
- Understanding how concurrent requests interact with shared application state.
- Identifying race windows between different stages of an operation.
- Understanding how timing and request execution order can affect application behavior.
- Recognizing how race conditions can lead to unexpected application states.

### 2. Limit Overrun Race Conditions
- Understanding how concurrent requests can bypass application limits.
- Exploiting race conditions to exceed restrictions such as purchase limits or transaction limits.
- Identifying situations where validation and state updates are not performed atomically.

### 3. Detecting and Exploiting Race Conditions
- Using Burp Repeater to investigate application behavior.
- Comparing sequential and parallel request execution.
- Identifying inconsistent response times and unexpected responses.
- Using Burp Turbo Intruder to send large numbers of requests concurrently.
- Understanding how HTTP/2 and single-packet attacks can improve request synchronization.

### 4. Hidden Multi-Step Sequences
- Identifying race conditions in workflows involving multiple requests.
- Understanding how applications process multi-step operations.
- Investigating hidden execution sequences and state transitions.
- Exploiting timing gaps between different stages of an operation.

### 5. Methodology for Identifying and Exploiting Race Conditions
- Identifying endpoints that modify shared application state.
- Understanding the application's business logic and request dependencies.
- Benchmarking normal application behavior.
- Identifying potential race windows.
- Testing requests individually, sequentially, and concurrently.
- Repeating attacks to account for timing variations.

### 6. Multi-Endpoint Race Conditions
- Identifying interactions between separate application endpoints.
- Understanding how two different operations can modify the same underlying state.
- Exploiting race conditions between operations such as adding items to a cart and checking out.
- Investigating how concurrent requests can bypass business logic restrictions.

### 7. Single-Endpoint Race Conditions
- Understanding race conditions involving multiple requests to the same endpoint.
- Investigating concurrent password reset operations.
- Identifying weaknesses in token generation and validation.
- Exploring how carefully timed requests can produce unexpected authentication behavior.

### 8. Session-Based Locking Mechanisms
- Understanding how applications serialize requests belonging to the same session.
- Investigating how session-based locking affects concurrent request execution.
- Testing requests originating from separate sessions.
- Understanding how locking mechanisms can influence the exploitability of race conditions.

### 9. Time-Sensitive Vulnerabilities
- Investigating vulnerabilities involving time-dependent cryptographic operations.
- Understanding how predictable inputs can affect security token generation.
- Exploiting timing behavior to identify token collisions.
- Understanding the importance of unpredictable, securely generated, and user-bound reset tokens.

## Tools and Techniques Used

- **Burp Suite Professional:** Intercepting, modifying, and analyzing HTTP requests.
- **Burp Repeater:** Sending requests sequentially and in parallel.
- **Burp Turbo Intruder:** Automating concurrent requests and investigating race windows.
- **HTTP/2:** Exploring synchronized request execution and single-packet attacks.
- **Burp Proxy:** Observing application traffic and identifying relevant endpoints.

## Key Takeaways

- Race conditions are fundamentally timing-dependent vulnerabilities.
- Application operations involving shared state should be implemented atomically.
- Security checks and state-changing operations must be properly synchronized.
- Testing requests individually may not reveal vulnerabilities that appear only under concurrent execution.
- Multiple endpoints can interact with the same underlying application state, creating unexpected attack opportunities.
- Session-based locking does not necessarily prevent race conditions across different sessions.
- Security-sensitive tokens must be unpredictable and properly associated with their intended users and operations.

## Conclusion

Completing this learning path provided practical experience in identifying and exploiting race conditions across different web application workflows.

It also improved my understanding of how concurrency, application state, session management, and request timing interact, and why testing business logic under concurrent execution is an important part of web application security testing.