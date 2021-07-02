---
title: "Pipeline Pattern in Go Part 1"
date: 2021-07-02T09:36:46+05:30
description: "yet another concurrency pattern in go"
---

I am going to divide this post into two parts, In the first part I will try to explain basic building blocks of pipelines and in second post I will try to build a general purpose library around this design.

I was recently reading [Concurrency In Go](https://learning.oreilly.com/library/view/concurrency-in-go/9781491941294/) and came across something called as pipeline processing pattern. Idea is you can break a logical functionality into stages. Each stage does its own processing and passes the output to the next stage to get processed. You can modify stages independent of one another, rate limit the stages and so on and forth.

## Pipeline basics

Consider these two functions

```go
func Multiply(value ,multiplier int) int { 
	return value*multiplier
}
```

```go
func Add(value,additive int) int { 
	return value+additive
}
```

What these function do is very simple, They just perform the arithmetic operation on number with a constant and return it. You can think of these as "stages" of pipeline. To complete the pipeline we can just combine both stages.

```go

ints := []int{1,2,3,4}

for _,v := range ints {
	fmt.Println(multiply(add(multiply(v,2),1),2))
}

```

We can observe here that stage consumes the input, processes it and returns the same type for further stage processing.


## Concurrent Pipelines


This simplistic model can now be extended to utilize go's channels and goroutines to perform the processing of the stages concurrently. 
Before we can do that we must have following entities in our pipeline.

- **Generator**, which would be responsible for producing the input for pipeline processing. You think of this as first stage of the pipeline.
  
- **Stages**, where the actual processing can be performed.
  
- **Canceller**, a mechanism to signal cancellation or end of processing by pipeline.


Let's start with generator

```go
func generator(done <-chan interface{}, integers ...int) <-chan int {
	intStream := make(chan int)
	go func() {
		defer close(intStream)
		for _, i := range integers {
			select {
			case <-done: return
			case intStream <- i:
			}
		}

	}()
	return intStream
}
```

This function simply spawns a goroutine which tries to produce values on a channel, and the function simply returns that channels. Values generated on this channel serves as the input for further stages. We're also passing done channel in the function to gracefully exit the generation, this is also known as [Poison Pill Pattern](https://java-design-patterns.com/patterns/poison-pill/)


Extending the same idea, we can rewrite multiply and add function to do the processing concurrently.


```go

func multiply(done <-chan interface{}, intStream <-chan int, multiplier int) chan int {
	multipliedStream := make(chan int)
	go func() {
		defer close(multipliedStream)
		for i := range intStream {
			select {
			case <-done: return
			case multipliedStream <- i * multiplier:
			}
		}
	}()
	return multipliedStream
}


func add(done <-chan interface{}, intStream <-chan int, adder int) chan int {
	addedStream := make(chan int)
	go func() {
		defer close(addedStream)
		for i := range intStream {
			select {
			case <-done:return
			case addedStream <- i * adder:
			}
		}
	}()
	return addedStream
}

```

**Sidenote:** Perhaps we can rewrite these functions to accept interface or function instead of writing the function twice which share most of the code, but I am trying to keep things simple here.

In order to build pipelines we can combine these stages of pipelines as

```go

	done := make(chan interface{})
	intStream := generator(done, 1, 2, 3, 4)

	pipeline := add(done, multiply(done, intStream, 2), 5)

	for i := range pipeline {
		fmt.Println(i)
	}
```

This piece of code will run stages concurrently in different pipes, keep passing their result to next stage via go channels. We can get the final results by reading last stage's channel. 


## More practical use case

Ok, but how about something more practical? We can implement a toy shopify scraper using this design. Aim is to crawl shopify app directory page and scrape the useful information.

Here's the basic design algorithm we can follow

- Goto app directory page e.g https://apps.shopify.com/browse/all?page=1 ![App Directory Page](https://i.imgur.com/noNqhT8.png)
  
- From each app directory page scrape URLs to different apps.
  
- Visit each app URL scrape app information and populate our struct.

  ![App Listing Page](https://i.imgur.com/NxPEgfB.png)

- Walk through the pagination to go to the next app directory page and repeat.

In pipeline terms, we need to build something like this

![Scraper](https://i.imgur.com/qTBtL19.png)

Lets define App struct

```go
type App struct {
    Rating     float64
    Review     int
    Name       string
    Category   []string
    AppLink    string
    PageLink   string
}
```

I am not writing the actual scraping code to keep it short and simple but scraping logic can easily be plugged in without changing code structure.

```go

type ShopifyScraper struct {}



// Generation Stage, This stage will generate app directory URLs.
func (s *ShopifyScraper) Generate(done <-chan bool) chan App {

	links := make(chan App)
	go func() {
		defer close(links)
		for i := 0; i < 5; i++ {
			select {
			case <-done: return
			case links <- App{PageLink: fmt.Sprintf("https://apps.shopify.com/browse/all?page=%d", i)}:
			}
		}
	}()
	return links
}

// App Directory Scraper Stage, This stage will visit directory page and scrape different app URLs from the page for further processing.
func (s *ShopifyScraper) AppDirectoryScraper(done <-chan bool, input <-chan App) chan App {

	apps := make(chan App, 5)
	go func() {
		defer close(apps)
		for app := range input {

			// Extract Different App URLs by scraping  using app.PageLink

			select {
			case <-done: return
			case apps <- app:
			}
		}
	}()
	return apps
}

// App Scraper Stage, This stage will visit app page and scrape app information
func (s *ShopifyScraper) AppScraper(done <-chan bool, input <-chan App) chan App {
	apps := make(chan App)
	go func() {
		defer close(apps)
		for app := range input {
			
			// Extract Different App Information by scraping by using app.AppLink
			
			select {
			case <-done: return
			case apps <- app:

			}
		}
	}()
	return apps
}


```

Then to finally link the stages together to form a pipeline.

```go

func (s *ShopifyScraper) Do() []AppInfo {

	var result []AppInfo
	done := make(chan bool)
	defer close(done)

	for i := range s.AppScraper(done, s.AppDirectoryScraper(done, s.Generate(done))) {
		result = append(result, i)
	}
	return result
}

```

In the next post we will try to build pipelines as a library which can take care of (1) Pipeline ordering (2) Concurrently executing same stage in multiple goroutines (3) Cancellation (4) Possibly error handing / retry 