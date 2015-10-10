# Go - practice of asynchronous communication

This post explores some aspects of asynchronous communication in golang. It covers channels, executions in the main OS thread and how to move blocking operation into a separate go routine.

* Channels and empty struct
* Direction of channels
* Executing in the main thread
* Moving blocking operations into separate thread

## Channels and empty struct

Goroutines allow you to run a piece of code in parallel to others. But to employ it usefully,
there are a few additional requirements - we should be able to pass data between goroutines and
we should be able to synchronize execution of these goroutines. Channels provide the way to do that,
and they work alongside goroutines. Sometimes it does not matter what is being sent into the
channel - it’s only a fact of sending what it matters. You could see :

```golang
done := make(chan bool)
/// [...]
done <- true
```

The size of the boolean is platform dependent, but quite frankly it’s not the case when we should worry
about it's size. There is a way to not sending anything into the channel at all, more precisely
send an empty struct into the channel.

```golang
func main() {
    done := make(chan struct{})
    // start another goroutine; when it completes signal on the channel
    go func() {
        // execute logic
        done <- struct{} // Send a signal; Values does not matter, using empty struct here
    }()
    <- done // wait for other goroutine to finish; discard sent value
}
```
It's also important to note that the main goroutine will be blocked until it receives data
from some other goroutine. Channel in this case is synchronous and blocking.


## Direction of channels

You can also specify the direction of data movement on a channel by indicating the direction around the `chan` keyword.

Let's look at the previous example again:

```golang
func main() {
    done := make(chan struct{})
    // start another goroutine; when it completes signal on the channel
    go func() {
        // execute logic
        done <- struct{} // Send a signal; Values does not matter, using empty struct here
    }()
    <- done // wait for other goroutine to finish; discard sent value
}
```
We only write data into the channel in the separate goroutine and we will only be reading
data from that channel in main goroutine. In this case it's useful and cleaner to
declare this channel as send-only and receive-only correspondingly

```golang
func main() {
    done := make(chan struct{})
    go func(done chan<- struct{}) { // declares it's a send-only channel
        // stuff
        done <- struct{}{} // notify channel before finishing goroutine
    } (done)
    <- done // waiting for goroutine to finish
}
```

Now, if we pass channel into the function like that it will become a send-only channel. But outside of this
function the channel will still remain bidirectional. It's also possible to transform channel to
a receive-only or send-only without passing it into a function:

```golang
done := make(chan struct{})
writingChan := (chan<- struct{})(done) // first parathesis are not necessary
receiveOnlyChan := (<-chan struct{})(done) // first paramthesis are necessary
```

Sometimes it could be handy to eliminate some of the mistakes at a compile time.


## Execution in the main thread
Some libraries, especially graphical frameworks/libraries like Cocoa, OpenGL, libSDL all require it's
called from the main OS thread or called from the same OS thread due to its use of thread local
data structures. Go's runtime provides LockOSThread() function for this. If this function is called
in the init method, this goroutine will be attached to a main OS thread. All the other
goroutines will be happily executing in the different threads in parallel.

To move our business logic execution into a separate thread, you can just send function into the main one. Here is an example:

```golang
package main

import (
	"fmt"
	"runtime"
)

func init() {
	runtime.LockOSThread()
}

func main() {
	fmt.Println("start main thread")
	done := make(chan bool)
	stuff := make(chan func())
	go func(done chan<- bool, stuff chan<- func()) {
		fmt.Println("start secondary thread")
		stuff <- func() {
			fmt.Println("1")
		}
		stuff <- func() {
			fmt.Println("2")
		}
		stuff <- func() {
			fmt.Println("3")
		}
		done <- true
	}(done, stuff)

	for i, exit := 1, false; !exit; i++ {
		fmt.Println("for exec: ", i)
		select {
		case do := <-stuff:
			fmt.Println("execute in main thread")
			do()
		case exit = <-done:
			fmt.Println("exiting main")
			break
		}
	}
}
```

## Moving blocking operations into a separate thread

Quite regularly we are dealing with blocking IO calls, the good thing is they are really easy to improve:

```golang
package main

import "os"

func main() {
	stop := make(chan struct{})
	done := make(chan struct{})
	write := make(chan []byte)

	go func(write <-chan []byte, stop <-chan struct{}, done chan<- struct{}) {
		for {
			select {
			case msg := <-write:
				os.Stdout.Write(msg)
			case <-stop:
				done <- struct{}{}
			}
		}
	}(write, stop, done)
	write <- []byte("Hello ")
	write <- []byte("World!\n")
	stop <- struct{}{}
	<-done
}
```
However, if more than one goroutines will be sending messages to one receiving channel,
they will be concurrently blocked. In this case it makes sense to use buffered channels.


Links:
