# SL: SSL Without Sema-chisms
## Introduction
Today, we will be doing a deep dive into SSL programming alone, without the use of sema-chisms, so it actually just becomes Syntax Language (SL) rather than Semantic/Syntax Language.

Without sema-chisms, SL can only:
- Recognize input tokens
- Push and pop the return stack
- Generate output and error tokens

This makes it mathematically equivalent to a **PUSH DOWN TRANSDUCER**, making the grammar class for CONTEXT-FREE LANGUAGES.

![[Pasted image 20230117164047.png]]

## The Base Language
SSL is a very terse language, compared to a langauge like C. It's hard to understand until you get used to it.

Most statements and operations are represented by single characters such as:

| Character | Information |
|------------|--------------|
|`:`| Declaration, used to identify parts of code|
|`{ }`|Loop statement, for enclosing looping statements|
|`>`|Loop exit, break from a loop statement|
|`[ ]`| Case/if statement, used for selecting choices|
|`\|`|Case/if alternative, used with the case if for specifying alternative choices, backslash is to escape the value|
|`@`|Call statement, to call a specific rule|
|`>>`|Return statement, return from a defined rule|

Comments use `%` to the end of line convention e.g.
```
@ruleCall % this is a comment
```

## Program Structure
SSL programs have two main sections: definitions and rules.

DEFINITIONS give the names of tokens, types and constants used in the program. By types, we mean user defined types, as there are no system types in SSL.

Rules are a set of subprograms defining the actions of the SSL program. Rules can call other rules as well as be recursive.

Execution begins with the first rule encountered in the program, hence can be said to be the main rule.

The below code block, contains the generic code for an SSL program:

![[Pasted image 20230117165248.png]]

We have our definitions section where we use the definition operator (`:`). We can define our set of input tokens using the `input` keyword. The same can be done for output tokens with the `output` keyword. 

We can define our user types with the `type` keyword, and define the interface for sema-chisms with the `mechanism` keyword.

The rules section begins with the `rule` keyword, which is then followed by declaration of the rules, and ended with the `end` keyword.

## SSL Rules
Rules are subprograms that are made up of a set of actions to run. There are two types of rules:
- Procedure rules ( which just run a set of actions)
  ```
  name: 
	  actions;
  ```

- "Choice" rules (run a set of actions and can return a result)
```
name >> type: % type is the user defined data type we are returning in the choice rule
	actions;
```

SSL actions correspond to statements in other langauges.

## Rules Call and Return
Rules are called with the `@` action (recall this is the call rule operation).
Rules return when it reaches the end of the rule or using the `>>` action (the return operation),

```
ProcedureDef:       % main rule, execution starts here
	@ProcedureHeader
	@ProcedureBody;

DoStuff:             % procedure rule, does not return anything
	@DoFirstThing
	>>
	@DoOtherThing;

Foo >> SymbolKind:   % choice rule that returns of type SymbolKind
	@Bar
	>> sVar;

DoNothing:      % empty procedural rule
	;
```

Note in the above code how each rule definition ends with a semicolon, (only one semicolon is required to terminate the rule definition)

`ProcedureDef` will be our main rule in this case, and it calls `ProcedureHeader` and `ProcedureBody` with the `@` keyword.

We also see an example of procedural rules with `DoStuff` and `DoNothing` as well as a choice rule `Foo` which returns a constant of type `SymbolKind`. This would be a symbol defined by the user.

## SL Actions
The SL Subset of S/SL has 8 actions, some of which we have already seen:

|Action|Symbol|Description|
|------|------|----|
|Call|`@`|Call|
|Return|`>>`|Return|
|Input|`<token name>`|Recognizes an input token of the name `token name` e.g. x|
|Emit|`.<token name>`|Generate the output token `token name` to the output string, the operator in this case is the `.`|
|Error|`#<token name>`|Generate an error token|
|Cycle|`{ }`| Repeat a sequence of actions|
|Exit|`>`|Exit a cycle
|Choice|`[ ]`| Choose between sets of actions, implements if/case/switch in SL|

### Input Action
The input action is implicit, and is done by specifying the token name (writing the token name as an action). The input action essentially requires the next token from the input stream to be the particular token we named. For example, if we wrote `pColon`, we are expecting the next input token to be the `pColon` input token.

In this way we can specify what we expect the next token to be, and if the next token from the input stream does not match the specified token, SSL generates an error and SYNTAX ERROR RECOVERY is invoked. This allows us to perform simple syntax checking on the token stream from the scanner/screener.

If the next input stream token matches the specified token, (and the call is outside of a choice operation (more on this later)), then that token is consumed from the input stream.

Tokens are specified in our definitions section in one of the three following ways:
- A symbolic name e.g. `pColonEquals`
- A string synonym e.g. `:=`, where we match against the text of the token in quotes
- A wildcard that matches any next input token (e.g. `?` or `*` )

An example of a token definition is shown below:
![[Pasted image 20230117172856.png]]

We see the above definitions defining tokens with string synonyms.

### Emit Action
The emit action allows us to emit the specified token into the output stream, this is done by putting a period character infront of the name of the token we want to push to the output stream.

Consider the following code:
![[Pasted image 20230117173149.png]]

Which is called to parse the following input stream (sourced from `3*5+7*8`):
![[Pasted image 20230117173211.png]]

Note: The whitespace above is not present in the actual token stream, it is added for clarity in understanding the steam of tokens

Let's go through this rule call and understand what is happening:
- First we call the `Term` rule
	- Consumes a `pInt` token from the input stream, which corresponds to `3`
	- It then emmits an `sIntLit` token into the output stream, which would correspond to the integer literal for 3 (which for notes's sake we can call `int(3)`)
	- Consumes a `*` token, (i.e. the asterisk character) from the input stream
	- Consumes another `pInt` token which corresponds to 5
	- Emits the `sIntLit` corresponding to int(5)
	- Emits an `sMult` token which tells the later stages that we are performing a multiplication
	- Our output stream thus far is `sIntLit(3) sIntLit(5) sMult`
	- Return to the calling rule
- Consumes the `+` token
- Calls `Term` rule
	- This consumes the 7, 8 and * in the input stream, emitting the integer literals for 7 and 8 as well as the `sMult` token for multiplicatiion, done in the same steps as the previous call
	- Our output stream is now `sIntLit(3) sIntLit(5) sMult sIntLit(7) sIntLit(8) sMult`
	- Return to the calling rule
- Emits the `sAdd` token

Our final output stream is:
```
sIntLit(3) sIntLit(5) sMult sIntLit(7) sIntLit(8) sMult sAdd
```

Which can (logically) be interpreted to:
```
3 5 * 7 8 * +
```

In summary, our rules above transformed the input stream:
```
pInt(3) * pInt(5) + pInt(7) * pInt(8) 
→ 
sIntLit(3) sIntLit(5) sMult sIntLit(7) sIntLit(8) sMult sAdd
```

and logically:
```
3 * 5 + 7 * 8 
→
3 5 * 7 8 * +
```

### Error Action
The error action is used to report errors from the SSL program. It works just like the output action but instead emits the token into the error stream, rather than the output stream. This is done with `#` prefix:
```
#eMissingSemicolon
#eTypeMismatch
```

### Cycle and Cycle Exit Actions
Cycles are the SSL version of loops:
```
{
	actions
}
```

Actions within the cycle are repeated until the cycle is exited ( with `>`) or the rule is exited ( using a return `>>`)

Cycles may be nested, with the exit action terminating only the innermost cycle.

![[Pasted image 20230117174525.png]]

### SL Choice Action
The choice action implements conditional flow of control, like an if/case/switch ststaments in other languages. It follows the generic format:

![[Pasted image 20230117174706.png]]

The choice begins with a selector word, which is checked against the specified labels. If the selector matches one of the labels, then its underlying actions will be run.

The selector can either be a choice rule (where in this case the selector value will be the constant returned from the choice rule) or nothing (absent). If the selector is absent, the next input token becomes the selector (i.e. SSL will check the next input token against the labels provided)

If the selector is the next input token (i..e. defined absent selector), then a matched label will consume that input token.

SSL also provides a default case, with  the `*` wildcard label. If the selector is the next input token and it matches against the default label, the input token will not be consumed.

That is to say, the next input token is only consumed if the selector is absent and the input token matches one of the specified labels.

If no default case/label is provided, then SSL generates an error and SYNTAX ERROR RECOVERY is invoked.

### Actions Example
Consider the following exampe:

![[Pasted image 20230117180815.png]]

If we consider the input `A(J) := 1`, we can follow the call:
- Begins with the main rule AssignOrCall
- Consumes `pIdentifier` which corresponds to `A`
- Calls the optionalSubscript rule
	- This starts with a choice that has no selector, hence we match against the next input token
	- Next input token `(` matches the case label, and therefore it is consumed
	- Case calls the Expression rule
		- `J` matches the `pIdentifer` label in the choice statement, and is consumed
		- Calls the optionalSubcript rule
			- `)` matches the default case in the rule, and is therefore not consumed
			- Nothing happens for this case so the rule returns
		- Expression returns
	- `)` is consumed by the input token requirement (follows after the Expression rule call)
	- optionalSubscript returns
- Choice selector in AssignOrCall, `:=` matches the first label and is therefore consumed (no selector means next input token is the selector)
- Calls Expression Rule
	- `1` matches the `pInteger` token and is therefore consumed (since no selector was provided)
	- Expresssion eturns
- `;` is consumed as following the procuedre requirement
- AssignOrCall ends execution

You can do this for the remaining examples on the screen.

### Actions Example #2
![[Pasted image 20230117202654.png]]

The above example shows the use of the choice rule as a selector. Our choice rule returns the type Boolean (which the user defined themselves).

Within the choice rule, we have next token matching choice ( no selector so matching against next token), that returns true if a comma is read and false is a bracket is read. (both true and false are user defined constants of the Boolean type)

The choice rule is called within someRule and placed in the selector position. The returned constant is consumed as part of the label matching and then executes required actions.

These two rules would be equivalent to:

![[Pasted image 20230117202946.png]]

## SSL Definitions
As said before, we use the definitions to specify tokens as well as type and constant declarations. These are done within the input, output and error section headers.

![[Pasted image 20230117203426.png]]

We see that token definitions can include the string synonym for them in line. Note that it is convention to begin error tokens with an `e` to indicate their use.

User defined types are returned by the choice rules, and are also used to communicate with semantic mechanisms.

In the definition, they specify an ordered set of named values, much like an enumerated type in C++.

![[Pasted image 20230117203618.png]]

As seen above, we begin a type declaration using the `type` keyword, and then defining the different constants that are a part of the user type.

## Implementation of SSL Types
The enumerated values we use for defining tokens as well as user type acceptable values are represented as INTEGER CONSTANTS in the host language.
The HOST LANGUAGE is the language that is used to understand and execute the behaviour specified by the SSL, we will cover more about them later. In our case, our host language is PT Pascal.

![[Pasted image 20230117204042.png]]

We can explicitly control the integer values used to represent the tokens by optionally providing a value as part of their declaration:

![[Pasted image 20230117204209.png]]

This has no effect on the SSL program, but constrains the implementation (host language) to encode the token using the given value, this is done usually for external reasons.




