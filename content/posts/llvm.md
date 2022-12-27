Goal of this post is to take a brief look into LLVM infrastructure and LLVM IR, then using that knowledge to build a toy brainfuck compiler that emits IR which can then further be compiled into native executable by LLVM tools. You can find the full source code in this [repository](https://github.com/spl0i7/gollvm-bf)

# What is LLVM? 

So what even is LLVM? LLVM Project is a compiler framework and collection of tools which can be used to build programming languages. In fact, LLVM is not a "single" project but is composed of multiple subprojects under the same umbrella which includes LLVM Core, Clang, LLDB, libc++, etc. By the way LLVM doesn't stands for anything, it's just LLVM. 

Let's try to understand what a compiler is before we go into any further topics. Compiler is a program that translates a program from one language to another, most commonly to native machine language but not always necessarily. It can also translate from one high level language to another, those are called transpilers. 

![design](https://i.imgur.com/eQjSgoG.png)

Typical compiler design includes 

- **Frontend** - Tokenizes the code, parses, checks for errors,  converts to AST (Abstract Syntax Tree) and optionally intermediate output called IR (intermediate representation)

- **Optimizer** - Does it's magic to optimize the code

- **Backend** - Generate the final output depending on the compilation target (os, architecture, etc)

This design has a advantage that each of these steps can be implemented independently of each other. For example we can write different frontend for different programming languages which all output same intermediate representation. Optimizer can be designed in such a way that it optimizes only IR which is going to be same for all the language. Similarly backend can be designed to operate on IR as input and output code which is dependent on target. This way we end up reusing different part of compiler for different programming languages without having to start from scratch for every new compiler. 

This is kind of how LLVM is designed. Frontend parses the source into LLVM IR, IR is fed through optimizer(s) which is then fed into code generator to generate target output. IR is thus the interface to the LLVM compiler framework. This property means that all you need to know to write a front end for LLVM is IR and LLVM tools can then take over to build the final executable.



## LLVM IR

LLVM IR is a low-level intermediate representation used by the LLVM compiler framework. LLVM has static single assignment form ([SSA](https://en.wikipedia.org/wiki/Static_single-assignment_form)). GCC also has IR called GIMPLE but it is not easy to work with as it's more tighly coupled to the compiler implementation. LLVM also has a great API(s) available in different programming langauges via libraries which makes it a delight to work with.

### IR Structure 

Let's take a highly simplified look into how LLVM IR is structured. In general, it takes the following form


![alt text](https://i.imgur.com/6cBlRWh.png)

#### Module
Module is the outermost container of the IR. Module contains target information, It contains symbol, Global variables, Function declarations, function definition and other things (not relevant to current discussion). Most commonly modules are created per source file. Target information just a long string which describes target for the IR.

#### Function
Function contains arguments, entry basic block and basic blocks.

#### Basic Block
Basic block contains labels, Instructions followed by termination instructor. Every basic block must end with a terminator. Terminator cannot appear anywhere else other than at the end of a Basic Block. These tell where the control of execution should go to when the reach the end of basic block. (most common one is `ret`)

Let's take a look at LLVM IR for the following C code snipped which squares and adds two integer

```c
int square_add(int a, int b) {

	a = a * a;
	b = b * b;
	return a + b;
	
}
```

```llvm
define i32 @square_add(i32 %a, i32 %b) {

entry:

	%a.addr = alloca i32
	%b.addr = alloca i32
	
	store i32 %a, i32* %a.addr
	store i32 %b, i32* %b.addr
	
	%0 = load i32, i32* %a.addr
	%1 = load i32, i32* %a.addr
	
	%mul = mul nsw i32 %0, %1
	store i32 %mul, i32* %a.addr
	
	%2 = load i32, i32* %b.addr
	%3 = load i32, i32* %b.addr
	
	%mul1 = mul nsw i32 %2, %3
	store i32 %mul1, i32* %b.addr
	
	%4 = load i32, i32* %a.addr
	%5 = load i32, i32* %b.addr
	
	%add = add nsw i32 %4, %5
	
	ret i32 %add

}
```
[Godbolt](https://godbolt.org/z/xfGnGeYEh) (I've cleaned up the example to make it easier to read)

That looks like C style assembly hybrid and that's kind of what it is. Let's try to go through what all this generated code means. There's a function declaration in first line which tells us type this function will be returning, then it's name `square_add` (I changed the name from generated output to explain better) and then finally arguments with their specified data types.

This function then contains a bunch of instructions to compute values from the gives arguments. In this example we see few of them `add`, `mul`, `ret`, `alloca`, `load`, `store`.

LLVM identifiers come in two types
- Global identifiers (functions, global variables) begin with the `'@'` character.
- Local identifiers (register names, types) begin with the `'%'` character. 

This in this example `%mul1 = mul nsw i32 %2, %3` is actually addition of two 32-bit integer which is stored in `%mul1` virtual register. There can be unlimited number of these virtual register unlike any real world CPU. LLVM automatically maps these registers to target register during IR to object compilation.

Let's take a look at these instructions

### `alloca` 

The `alloca` instruction is used to allocate memory on the stack of the current function which is then automatically freed when this function returns. This is exactly same as how local variable behave in C like programming languages


```
%a.addr = alloca i32
```

This instruction allocates space for a 32-bit signed integer on the stack. The pointer is stored in the register `%a.addr`.

### `store` 

The `store` instruction is used to write a value to memory

```
store i32 %mul1, i32* %b.addr
```

Here we tell LLVM to store a 32-bit integer from `%mul1` register into the register `%b.addr` which is a pointer to 32-bit integer. 

### `load` 

The `load` instruction is used to read from memory.

```
%2 = load i32, i32* %b.addr
```

In the example above, we load 32-bit integer from the memory address `b.addr`. This value is stored into the register `%2`.

### add / mul 

`add`, `mul` instruction are used to perform arithmetic operation between two registers.

```
%add = add nsw i32 %4, %5
```

In this example we add two 32-bit integers in `%4` and `%5` register and store its result in `%add` register.

### ret 
The ‘`ret`’ instruction is used to return control flow (and optionally a value) from a function.

```
ret i32 %add
```

In this example, we return 32-bit integer stored in `%add` register back to caller. 


### Hello world in IR 

```
@msg = constant [12 x i8] c"hello world\00"

declare i32 @puts(i8*)

define i32 @main() {
    %str = getelementptr [12 x i8],[12 x i8]* @msg, i64 0, i64 0

    call i32 @puts(i8* %str)
    ret i32 0
}

```

Writing a hello world is not too different from what we'd do in C. First we declare a i8 array of size 12 and initialize it with a value "hello world" with null termination.

We declare external `puts` function which will be resolved in linking step when this function will be mapped to c library by the linker.

Then we define main function which returns integer very much like our usual C programs. On the first line we use the infamous GEP instruction to get the address to the first byte of our array. Once we get that, we store the pointer in `%str` register and call the external `puts` function and return from the function with value `0`.

The `getelementptr` a/k/a `GEP` instruction is used to get the address of a subelement of an aggregate data structure. It's like `lea` instruction in x86 assembly. The first argument is always a type used as the basis for the calculations. The second argument is always a pointer or a vector of pointers, and is the base address to start from. The remaining arguments (if any) are indices that indicate which of the elements of the aggregate object are indexed.

To run this program save the code in `hello-world.ll` file and compile it with `clang`

```
$ clang hello-world.ll -O3 -o hello-world
```

Then finally run the compiled program to see it in action, you should hopefully see `hello world` get printed out.
```
$ ./hello-world
```

Now we can move onto implementing brainfuck compiler

# What is BrainFuck?

Brainfuck is a programming language which just has 8 instructions. A brainfuck program contains these 8 characters and optionally extra characters which is usually ignored as comments.  The instrunctions are executed one by one starting from the first instruction in the command till the end.

The model has a one-dimensional array of 30,000 cells initialized to 0, a data pointer which points as index in this array of cell, instruction pointer, an io stream to read input and write output.

### Commands

The eight language commands each consist of a single character:

- `>` Increment the data pointer (to point to the next cell to the right).
- `<` Decrement the data pointer (to point to the next cell to the left).
- `+` Increment (increase by one) the byte at the data pointer.
- `-` Decrement (decrease by one) the byte at the data pointer.
- `.` Output the byte at cell pointed by the data pointer.
- `,` Accept one byte of input, and store it in the cell pointing by the data pointer.
- `[` If the byte at the data pointer is zero, then instead of moving the instruction pointer forward to the next command, jump it forward to the command after the _matching_ `]` command.
- `]` If the byte at the data pointer is nonzero, then instead of moving the instruction pointer forward to the next command, jump it _back_ to the command after the _matching_ `[` command.

Since the language is very simple, writing an interpreter is not too difficult. You can find link to my interpreter [here](https://github.com/spl0i7/gollvm-bf/blob/main/interpreter/main.go)I will use the design of interpreter to write the compiler


# Crafting the Compiler

To begin, we'll first create a main function which can read the program source file using command line argument. Then we will declare a  LLVM "module" and try to go through each instruction in our language spec using a switch-case statement. In those cases we'll implement instruction specific IR code in the following sections and then finally print the module in the string format which is going to emit the IR to stdout.

```go

func main() {  
  
   file, err := os.Open(os.Args[1])  
   if err != nil {  
      log.Fatalln(err)  
   }  
  
   program, err := io.ReadAll(file)  
   if err != nil {  
      log.Fatalln(err)  
   }  
   
   mod := ir.NewModule()  
  
   for i := 0; i < len(program); i++ {  
      switch program[i] {  
      case '+':  
      case '-':  
      case '<':  
      case '>':  
      case '.':  
      case ',':  
      case '[':  
      case ']':  
      }  
   }  
  
   fmt.Println(mod.String())  
}
```

But before implementing instructions, we need some boilerplate code in place. We need to declare few function from the C standard library which will be used in this program for input/output and initializing the cell memory as 0.

```go

// declare C libary functions
getchar := mod.NewFunc("getchar", types.I8) 

puts := mod.NewFunc("putchar", types.I32, ir.NewParam("p1", types.I8))

memset := mod.NewFunc("memset", types.Void, ir.NewParam("ptr", types.I8Ptr), ir.NewParam("val", types.I8), ir.NewParam("len", types.I64))

// declare main
entryPoint := mod.NewFunc("main", types.I32)

// block inside main where IR instructions will live
builder := entryPoint.NewBlock("")
```

- `getchar()` -  function reads a single character from stdin and returns that character to the caller. This will be used to read from stdin to cell memory.
- `putchar()` - function writes a character to the stdout. This will be used to output the character to stdout in a memory cell.
- `memset()` - function writes `len` bytes of `val` to the string `ptr`. We will use this to initialize the cell memory to 0 at the start of the program.

Once we have these basic functions, we declare the `main` function for our program, and we add an unnamed block in our source code. This block is where rest of our code will live. 

Next we allocate space for a byte array of 30,000 cells, allocate space for data pointer which will be used as index for that array. Then we initialize both with 0. For array, we call the `memset()` and set the value to zero. For the pointer variable we just store 0 constant there.

```go
// type of array:  [30000 x i8]
arrayType := types.NewArray(MemSize, types.I8)

// create an array
cellMemory := builder.NewAlloca(arrayType)  

// allocate space for index in the memory cell
dataPtr := builder.NewAlloca(types.I64)  

// initialize the cell into zero
builder.NewStore(constant.NewInt(types.I64, 0), dataPtr)

// call memset: memset(&array, 0, 30000)

builder.NewCall(memset,  
   builder.NewGetElementPtr(arrayType, cellMemory, constant.NewInt(types.I64, 0), constant.NewInt(types.I64, 0)),  
   constant.NewInt(types.I8, 0),  
   constant.NewInt(types.I64, MemSize),  
)  

```


This forms the basic boilerplate we can now start implementing the language instructions.

##  Implementing '+'
To implement `+` instruction we need to get the cell pointed to by `dataPtr` index. Add one to it and then store it back in the array.

```go

case '+':  

// get the dataPtr th index in the array

   ptr := builder.NewGetElementPtr(arrayType, cellMemory, constant.NewInt(types.I64, 0), builder.NewLoad(types.I64, dataPtr))

// add 1 to it
	added := builder.NewAdd(builder.NewLoad(types.I8, ptr), constant.NewInt(types.I8, 1))  

// store it back in the array
	builder.NewStore(added, ptr)
```

##  Implementing '-'
This one is same as `+`  but we just decrement the value instead of adding.

```go

case '-':  

// get the dataPtr th index in the array

   ptr := builder.NewGetElementPtr(arrayType, cellMemory, constant.NewInt(types.I64, 0), builder.NewLoad(types.I64, dataPtr))

// subtrace 1 to it
	added := builder.NewAdd(builder.NewLoad(types.I8, ptr), constant.NewInt(types.I8, -1))  

// store it back in the array
	builder.NewStore(added, ptr)
```

##  Implementing '>'
This instruction is also very straightforward. We just load the `dataPtr` into a register, add one to it. Store this new value back in the `dataPtr` memory location.

```go

case '>':  

// increment the datapointer by one
   t1 := builder.NewAdd(builder.NewLoad(types.I64, dataPtr), constant.NewInt(types.I8, +1))  

// store the new value in the dataPtr
   builder.NewStore(t1, dataPtr)
```

##  Implementing '<'
This one is same as `>`, we just decrement the `dataPtr` by 1 instead of adding.

```go

case '<':  

// decrement the datapointer by one
   t1 := builder.NewAdd(builder.NewLoad(types.I64, dataPtr), constant.NewInt(types.I8, -1))  

// store the new value in the dataPtr
   builder.NewStore(t1, dataPtr)
```
##  Implementing '['

For `[` instruction we use a stack to keep track of current loop block (just like in the interpreter implementation). If the cell is zero we jump to the end of the loop otherwise we jump to the body of the loop.

Note that we've created two new blocks to denote body and end of the loop by creating new basic block. We did not name them explicitly (they will automatically be named). 

```go

case '[':
// get a pointer to cell pointed by dataPtr index
	ptr := builder.NewGetElementPtr(arrayType, cellMemory, constant.NewInt(types.I64, 0), builder.NewLoad(types.I64, dataPtr)) 
	
// load this cell value in ld
	ld := builder.NewLoad(types.I8, ptr)  
	
// check if this cell's vaue is equal to 0 or not.
	cmpResult := builder.NewICmp(enum.IPredNE, ld, constant.NewInt(types.I8, 0))
	
	
// create new basic block(s) to store the loop's body and end of loop
	wb := Loop{  
	   Body: entryPoint.NewBlock(""),  
	   End:  entryPoint.NewBlock(""),  
	}  
	
// push it to stack so that we remember when we encountered '['
	st.Push(wb)
	
// If cell is not zero then goto body of the loop otherwise goto end  
	builder.NewCondBr(cmpResult, wb.Body, wb.End) 
	
// start writing subsequent statements in to the body of the loop
	builder = wb.Body


```
## Implementing ']'

For `]` instruction we pop from stack to get the last occurrence of `[` instruction because if current cell value is not zero then we have to jump to there. Ofcourse we expect stack to be not empty and if it is empty it means that we have excess number if `]` compared to `[`
```go

case ']':
// pop from the stack, to get the last encountered '['
	front, ok := st.Pop()  
	
// if the stack is empty then we have unbalanced number of ']'
	if !ok {  
	   log.Fatalln(ErrUnexpectedBracket)  
	}  
	
// get a pointer to cell pointed by dataPtr index
	ptr := builder.NewGetElementPtr(arrayType, cellMemory, constant.NewInt(types.I64, 0), builder.NewLoad(types.I64, dataPtr)) 
	
// load this cell value in ld
	ld := builder.NewLoad(types.I8, ptr)  
	
// compare if cell is zero?
	cmpResult := builder.NewICmp(enum.IPredNE, ld, constant.NewInt(types.I8, 0))  
	
// if cell is not zero then goto body of the loop otherwise goto end
	builder.NewCondBr(cmpResult, front.Body, front.End)  
	
// write subsequent statement to end of the loop
	builder = front.End

```

## Implement ','
This just involves calling `getchar()` and storing the result in the array at an index pointed by `dataptr`

```go

case ',':  
// call getchar()
   char := builder.NewCall(getchar)  
   
// get pointer to the element in the array
   ptr := builder.NewGetElementPtr(arrayType, cellMemory, constant.NewInt(types.I64, 0), builder.NewLoad(types.I64, dataPtr)) 
   
// store the character in that pointer
   builder.NewStore(char, ptr)
```

## Implement '.'

```go

case '.':  
// get the pointer to current cell in the array
   ptr := builder.NewGetElementPtr(arrayType, cellMemory, constant.NewInt(types.I64, 0), builder.NewLoad(types.I64, dataPtr)) 

// call putchar()
   builder.NewCall(puts, builder.NewLoad(types.I8, ptr))
```

This covers all the instruction in the language spec. Now we just finally we add a terminator instruction to end our main function.

```go
builder.NewRet(constant.NewInt(types.I32, 0))
```

Full source code [here](https://github.com/spl0i7/gollvm-bf/blob/main/compiler/main.go)

# Benchmarking Performance

| program name                 | time(compiled with -O3) | time(interpreter) |
|------------------------------|-------------------------|-------------------|
| mandelbrot fractal (mb.bf)   | 620.74 millis           | 36.86 secs        |
| tower of hanoi (hanoi.bf)    | 11.47 millis            | 31.85 secs        |
| hello world (hello-world.bf) | 6.19 millis             | 213.93 millis     |

As expected compiler version is orders of magnitude faster than the interpreter. One interesting thing I observed in the tower of hanoi program was that optimizer basically ran the whole program at the compile time and just printed the final state of the tower with `-O3` flag which of-course is incredibly fast. We cannot even see disk moving if we use `-O3` flag which is mind-blowing level of optimization.

# Closing thoughts

I think LLVM is an amazing framework and makes building compiler as simple as it can get. Go's LLVM library is also very ergonomic to work with but many function just panic instead of returning an error properly which is not very pleasant especially because most tricky part of writing a compiler is giving well-formed error messages to the user. Apart from this, it was enjoying experience to use this library.


# References

- http://aosabook.org/en/llvm.html
- https://esolangs.org/wiki/Brainfuck
- https://llvm.org/docs/LangRef.html
- https://github.com/llir/llvm
- https://mukulrathi.com/create-your-own-programming-language/llvm-ir-cpp-api-tutorial/
- https://blog.yossarian.net/2020/09/19/LLVMs-getelementptr-by-example
- https://www.youtube.com/watch?v=m8G_S5LwlTo

