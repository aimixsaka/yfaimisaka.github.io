# Go Concurrency


## $ Before Write

Write this article to record the practical experience of go concurrency

### $ Content

- When deadlock occurs, consider whether the declared `chan` has been initialized with `make`
- Always send and receive values in `chan` in **different** `goroutine`, don't try to send and receive values in main
- If `chan` is **unbuffered**, always **first** run a `goroutine` to send or receive values, then** in main (or other `goroutine`) to receive or send
- To send and receive values from `chan` in `main` at the same time, consider using a **buffered channel**
- Always consider the deadlock problem caused by two `chan`s that send and receive each other (for example, two `chan`s are sent to each other at the same time, and no one receives them at this time), you can put a send operation in a separate `goroutine` to solve
- Blocking of other `goroutine` will not affect the `main goroutine` at all
