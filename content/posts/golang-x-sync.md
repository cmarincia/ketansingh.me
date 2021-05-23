---
title: "The other sync package"
date: 2021-05-22T19:48:19+05:30
description: "rendezvous with the forgotten go package"
---

I am sure everyone familiar with Go has come across the concurrency primitives at some point which mainly includes goroutines, channels and the `sync` package which provides us with `Mutex`, `RWMutex`, `WaitGroup`, etc. primarily used for synchronization. In this post I want to draw attention to another useful but less widely known sibling of the `sync` package, `golang.org/x/sync`. This particular package is relatively small enough that I'll be able to demonstrate it this post.

# [errgroup](https://pkg.go.dev/golang.org/x/sync@v0.0.0-20210220032951-036812b2e83c/errgroup)

`errgroup` provides synchronization, error propagation, and Context cancelation for groups of goroutines working on subtasks. In other words you can use this in scenarios where typically `sync.WaitGroup` is used but this one also takes care of passing a `Context` into the subtasks and handling errors automatically for you.


Consider following example which uses `sync.WaitGroup`


```go
func processTasks() {

	tasks := 10
	wg := sync.WaitGroup{}
	wg.Add(tasks)

	for i := 0; i < tasks; i++ {
		go func(i int) {
			defer wg.Done()
			if err := requestAPI(i); err != nil {
				print(err.Error())
			}
		}(i)
	}

	wg.Wait()

}

func requestAPI(i int) error {
	var err error
	url := fmt.Sprintf("https://jsonplaceholder.typicode.com/todos/%d", i)
	resp, err := http.Get(url)
	if err != nil {
		return err
	}
	_, err = io.Copy(ioutil.Discard, resp.Body)
	return err
}
```

While this works alright, code looks clunky espicially with respect to error handling. Now if we refactor it to use `errgroup` it takes the following form 

```go
func processTasks() {

	grp, _ := errgroup.WithContext(context.TODO())
	for i := 0; i < 10; i++ {
		currentIdx := i
		grp.Go(func() error {
			return requestAPI(currentIdx)
		})
	}

	if err := grp.Wait(); err != nil {
		print(err.Error())
	}

}

func requestAPI(i int) error {
	var err error
	resp, err := http.Get("https://jsonplaceholder.typicode.com/todos/" + fmt.Sprint(i))
	if err != nil {
		return err
	}
	_, err = io.Copy(ioutil.Discard, resp.Body)
	return err
}
```

We get reference to `Context` which if we want can use it to cancel the tasks or time them out if needed. Only caveat to watch out here is that the first call to return a non-nil error cancels the group and its error will be returned by `Wait`.


# [semaphore](https://pkg.go.dev/golang.org/x/sync@v0.0.0-20210220032951-036812b2e83c/semaphore#section-sourcefiles)

This package provides a weighted semaphore implementation. If you have worked with go channels before you will know that buffered channels kind of behaves like a semaphore. Capacity of the buffered channel is the number of resources we wish to synchronize, length of the channel is the number of resources current being used and capacity minus the length of the channel is the number of free resources. However in the case of buffered channels everything is equal weight and it becomes non-trivial to implement it in scenarios where a goroutine might pick up "heavy" task and you have to rate limit it accordingly. This is where semaphore becomes very useful


```go
func processTasks() {

	sem := semaphore.NewWeighted(4)
	wg := sync.WaitGroup{}
	wg.Add(10)

	go func() {
		for i := 0; i < 5; i++ {
			if err := sem.Acquire(context.TODO(), 1); err != nil {
				panic(err)
			}
			time.Sleep(time.Second * 1)

			sem.Release(1)
			wg.Done()
		}
	}()

	go func() {
		for i := 0; i < 5; i++ {
			if err := sem.Acquire(context.TODO(), 3); err != nil {
				panic(err)
			}

			time.Sleep(time.Second * 3)
			sem.Release(3)
			wg.Done()

		}
	}()

	wg.Wait()

}
```



# [singleflight](https://pkg.go.dev/golang.org/x/sync@v0.0.0-20210220032951-036812b2e83c/singleflight)

`singleflight` provides a duplicate function call suppression mechanism, which means if multiple goroutines call a same function concurrently this package ensures that that only one execution is in-flight for a given key at a time. If a duplicate comes in, the duplicate caller waits for the original to complete and receives the same results.

```go

func processTasks() {


	sf := singleflight.Group{}
	wg := sync.WaitGroup{}
	wg.Add(5)
	for i := 0; i < 5; i++ {
		go func() {
			defer wg.Done()
			value, _, _ := sf.Do("my_task", heavyTask)
			println(value.(string))
		}()
	}

	wg.Wait()

}


func heavyTask() (interface{}, error){
	println("heavyTask()")
	time.Sleep(time.Second * 5)
	return "done", nil
}

```
You will observe that this prints `"heavyTask()"` only once even though there are 5 goroutines calling the same function. This is particularly useful when interacting with the "slow" function which are being called concurrently such as with database, reading files or making HTTP calls.


There's another member of this package called `syncmap`, which has already been included in the `sync` package. So, I decided not to cover it here.