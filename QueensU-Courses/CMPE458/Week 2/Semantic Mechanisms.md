# Semantic Mechanisms
## Introduction
SL is dataless, therefore to add more functionality we need the ability to manipulate values and make decisions based on those values.

SEMANTIC OPERATIONS provide this ability, and they are grouped into SEMANTIC MECHANISMS.

Semantic mechanisms (sema-chisms) are usually designed as  abstract data types, that is they provide operations on particular single conceptual data structure.

SSL programs are only aware of the sema-chism interface, as it is defined within the program, but the implementation of the sema-chism is not visible to the program.

## Defining Sema-chisms
### Defining their interface
The interface of a sema-chism is defined in the SSL program with the use of the `mechanism` keyword. For example, we can define a semantic mechanism for a stack of counters:

![[Pasted image 20230119151656.png]]

Under the mechanism definition, we specify all the semantic operations for the mechanism. 
The convention for naming operations is to:
- Begin with `o` prefix to indicate it is an operation
- Then prefix the name of the mechanism to indicate which mechanism a given operation belongs to

![[Pasted image 20230119152341.png]]

### Semantic Operations
Mechanisms can have two kinds of operations:
- Update operations: Procedural operations that just run a set of actions
- Choice operations: Functional operations that run actions but addditionally return a value

Update operations are defined by simply listing the name of the operation e.g.
```
oCountPop
oCountIncrement
```

Choice operations haev the name definition but include the type returned using the return operation symbol:
```
oCountChoose >> Number
```
This says that the operation `oCountChoose` returns `Number` , a user defined type.

Either kind can have a single parameter of a given type. For example `oCountPush(Number)` is an update operation that receives the defined user type Number.

Semantic operations are invoked in the program by simply using their name as an action in a rule. (This contrasts rule calls which begin with the `@` keyword).

### *Aside on the need for a counter mechanism*
A counter mechanism is useful in compilation as we may need to know how many parameters a function has, as well as parsing the dimensionality of an array (which can later be used for syntax checking).

The Count mechanism controls a stack of counters defined
- `oCountPush` pushes a new counter onto the stack initialized to start at `Number`
- `oCountIncrement` and `oCountDecrement` increment and decrement the top most counter on the stack
- `oCountPop` pops a counter from the stack
- `oCountChoose` pops the counter from the stack while returning its current value

Note that the defined `Number` type will have to have token definitions for every number we would like to use i.e.
```
type Number:
	zero = 0
	one = 1
	two = 2
	...
```

This may seem cumbersome, but its not that big of a deal as we only use a small set of numbers in parsing: 0,1,2 and 8.

## Example of Semantic Operations
The below example shows the parsing of an array's dimensions defined in Quby. Array definitions in the language are not limited to start at 0, and specify a range using the `..` notation, i.e. `1..4` says to define an array from index 1 to 4.

![[Pasted image 20230119153538.png]]

The ArrayType rule is called when we consume the array keyword token, this rule is called to parse the dimensions of the defined array.

Going through the lines of the rule:
- `oTypeEnterKind` is a semantic operation used to store the type of object we are currently parsing, it receives one parameter which would be type tokens the user defined. In this case, we pass it `tyArray`
- We then consume the `[` from the line
- We call `oCountPush` to push a new counter onto the stack, initialized to the defined constant `zero`
- Next we have a loop:
	- First we call the Expression rule to consume the start dimension number of the array: `1`
	- Then we consume the dot notation `..`
	- Then we call the Expression rule to consume the end dimension number of the array: `4`
	- We then increment the top most counter on the counter stack with the `oCountIncrement` operation. This shows that the counter is counting how many dimensions the array has
	- Next we have a choice operation with no selector, meaning that it matches and consumes the next token in the input
		- If the next input token matches a comma `,` , then we-reloop
		- Otherwise, we break from the loop
	- This choice allows us to continuously parse more dimension definitions for a given array, since we can define arrays with as many dimensions as we want
	- In this case, the comma is matched which causes the code to reloop
	- The Expression rules and the `..` input consumption parse the next dimension of the array `2..5`. We also increment the counter to keep track of the array dimensions
	- We encounter the choice again, but this time it breaks as no comma is present
- `oTypeEnterSubscriptCount` takes the value from the counter on the stack and enters it into our symbol table. This allows us to perform syntax checking on the indexing of the array
	- If the array has 2 dimensions, syntax checking will block `arr[2][3][5]`
- We use `oCountPop` to pop the counter since we done with it
- Finally, we call the ArrayType rule to handle the array element type

## A Complete SSL Program
Now that we have learnt the basis of SSL, we can understand a full SSL program for the scanner

![[Pasted image 20230119155351.png]]

We have our defined input tokens, output tokens and error tokens, as well as one mechanism for a character buffer.
- The token `cIllegal` is used to capture illegal characters for the soure code, the actual characters this matches will be defined in another file

Our main rule is the Scanner, which first calls SkipBlanks to skip the whitespace in the code. Then it parses letters or digits with a choice selector and calling the appopriate rule (ScanIdentifier for letters, ScanInteger for digits). If it encounters a plus or minus, it emits their associated token into the output stream.
- SkipBlanks indicates that operation symbols can be placed right beside each other, as it implements a choice loop with the use of `{[ ... ]}`

ScanIdentifier and ScanInteger use a loop to continously scan characters from the input stream. Note that ScanIdentifier has a label that can match against two tokens (`cLetter` and `cDigit`) with the use of a comma. This allows variable names to include digits as well.

They also continue to save the characters read into the buffer with the `oBufferSave` operation and at the end emmit a `pIdentifier` and `pInteger` token respectively.