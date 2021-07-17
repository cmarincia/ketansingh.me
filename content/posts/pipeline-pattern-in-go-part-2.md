---
title: "Pipeline Pattern in Go Part 2"
date: 2021-07-16T21:03:54+05:30
description: "yet another concurrency pattern in go (continued)"
---

This post is continuation of older post [Pipeline Pattern in Go Part 1](https://ketansingh.me/posts/pipeline-pattern-in-go-part-1/).
We will try to generalize the pipeline pattern into a library such that it can be used for different use-cases without having to repeat the whole thing everywhere.

P.S. You can find source [here](https://github.com/spl0i7/pipeline)
, You can also see it action on [playground](https://play.golang.org/p/2tPEV8MPoq7)


## Design Principle

It's important to quantize what we want out of this library, here's my thought process

- Stages in pipeline should receive same type and give back one or more values of same type.
- While initializing the library, we should be able to tell order of stages that is if we have stages `{S1, S2, S3}` message should be processed in order
`{S1 -> S2 -> S3`.
- We should be able to specify concurrency of the stages, that is if we have stages `{S1, S2, S3}` and `S2` is slow (Maybe it performs a heavy IO operation) then 
we should be able to create have multiple instances of this pipe `{S1 -> {S2, S2, S2} -> S3}`.
- There should be a start and stop mechanism for a pipeline.
- Ideally stages should be stateless but that depends on the programmer needs, but it's important if we want concurrency.
- Error should be retried (I will leave this as exercise for the reader)

## Let there be stage


```go

type Message interface {}

type Stage interface {
    Process(message Message) ([]Message, error)
}

```

Here we have defined two interfaces

- `Message` represents individual messages which stages will be passing among themselves. 
   
- `Stage` represents different stages which process a `Message` and can produce one or more message.

## Let there be pipeline

```go
type PipelineOpts struct {
    Concurrency int
}


type Pipeline interface {
    AddPipe(pipe Stage, opt *PipelineOpts)
    Start() error
    Stop() error
    Input() chan<- Message
    Output() <-chan Message
}

```


`Pipeline` is the container for collection of stages and will be responsible for moving the messages. 

- `AddPipe()` will add a `Stage` and link them in same order
- `Start()` will start the `Pipeline`
- `Stop()` will stop the `Pipeline` and will also wait until existing messages get processed (Perhaps we can have `ForceStop()` as well here?)
- `Input(), Output()` will be two ends of pipeline


Here's Pipeline represent visually 
![Pipeline](https://i.imgur.com/IqmwpO3.png)

- `Input()` is the input channel to first stage and then first stage's output is second stage's input and so on, That's how all the stages will be interconnected internally.
- `Output()` is the output channel of last stage.
- There could be multiple instances of a stage, they all will be connected by the same channel in a Single Producer Multiple Consumer (SPMC) manner or Multiple Producer Multiple Consumer(MPMC) depending on previous stages.



Okay so lets implement a Pipeline....err We need one more entity before we can do that


## Let there be workers

Why do we need this? As you can see we can have multiple instances of a stage. Other way to put this is that
multiple goroutines can execute a stateless stage. We need `StageWorker` to start goroutines for handling a stage, stop those goroutines when told so and drain all the channels so that all the buffered messages are processed.


```go
type StageWorker struct {
    wg          *sync.WaitGroup
    input       chan Message
    output      chan Message
    concurrency int
    pipe        Stage
}

func (w *StageWorker) Start() error {

    for i := 0; i < w.concurrency; i++ {
        w.wg.Add(1)

        go func() {
            defer w.wg.Done()
            for i := range w.Input() {
                result, err := w.pipe.Process(i)
                if err != nil {
                    log.Println(err)
                    continue
                }
                for _, r := range result {
                    w.Output() <- r
                }
            }
        }()
    }

func (w *StageWorker) WaitStop() error {
    w.wg.Wait()
    return nil
}
```

`Start()` here does few things 

1) Keeps count of running goroutine via `sync.WaitGroup`.
   
2) Calls the `Process()` for `Stage` and then passes the result to output channel for other stages.

`WaitStop()` just waits for goroutines to die and then returns



Our pipeline now looks like this, where stages are inside `StageWorker` container
![Pipeline](https://i.imgur.com/uxldG9i.png)



## Concurrent Pipeline

`ConcurrentPipeline` maintains a bunch of `StageWorker`, which is equal to number of stages


```go
type ConcurrentPipeline struct {
    stageWorkers []StageWorker
}

```


`AddPipe()` takes stage and options which for now just contains concurrency but can also contain other fields such as retry mechanism and so on.

This function links up different stages together via channels such that output of previous stage is an input to current stage.

```go
func (c *ConcurrentPipeline) AddPipe(stage Stage, opt *PipelineOpts) {

    if opt == nil {
        opt = &PipelineOpts{Concurrency: 1}
    }

    var input = make(chan Message, 10)
    var output = make(chan Message, 10)

    for _, i := range c.stageWorkers {
        input = i.Output()
    }

    worker := NewStageWorker(opt.Concurrency, stage, input, output)
    c.stageWorkers = append(c.stageWorkers, worker)
}
```

As mentioned previously `Output()` represent output channel of last stage, used for consumption of final output.

```go

func (c *ConcurrentPipeline) Output() <-chan Message {
    sz := len(c.stageWorkers)
    return c.stageWorkers[sz-1].Output()
}
```

Similarly `Input()` is input channel of first stage, typically used for starting a pipeline processing.
```go

func (c *ConcurrentPipeline) Input() chan<- Message {
    return c.stageWorkers[0].Input()
}
```

`Start()` Signals underlying StageWorkers to start.

```go
func (c *ConcurrentPipeline) Start() error {

    if len(c.StageWorkers) == 0 {
        return ErrConcurrentPipelineEmpty
    }

    for i := 0; i < len(c.stageWorkers); i++ {
        g := c.stageWorkers[i]
        g.Start()
    }

    return nil
}
```

`Stop()` closes the channels, causing goroutines inside StageWorkers stop and waits for them to exit gracefully.

```go
func (c *ConcurrentPipeline) Stop() error {

    for _, i := range c.stageWorkers {
        close(i.Input())
        i.WaitStop()
    }

    sz := len(c.stageWorkers)
    close(c.stageWorkers[sz-1].Output())
    return nil
}
```


Now that all the building blocks are in place, usage is fairly simple. You can check out the [example here](https://github.com/spl0i7/pipeline/blob/master/example/example.go).