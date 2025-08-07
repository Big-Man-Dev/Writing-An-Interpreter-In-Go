
# Writing An Interpreter In Go
<p align="center">
  <img src="https://interpreterbook.com/img/cover-cb2da3d1.png" />
</p>

## About this Project!

This is a project I worked on using Thorsten Ball's book Writing an Interpreter In Go.
This was very fun experience and gave me a lot more knowledge to the inner workings of other interpreted langauges like Python. For example people will often say that everything in Python is an Object. But few people understand what that even means. This project helped with that understanding and so much more. Let's dive into the features of our Interpreter called Monkey,

### Monkey's Features
    1) A C like syntax
    2) Variable Bindings 
    3) Integers, booleans, strings, arrays, and hashes.
    4) Built In methods
    5) Arithmetic
    6) higher-order functions and first-class functions 

## Lets program some Monkey!
In Monkey variables are implicitly typed. This means the type is assigned based on what data you provide, much like Python! lets declare some variables.

``` go
let x = 10;
let name = "Monkey";
let BooleanValue = false;
let myArray = [1,2,3,4];
let myMap = {"name" : "Monkey"}
// Writing any of the variables in the repl will result
// in the value being desplayed like below
x // => 10
myArray[1] // => 2
myMap["name"] // => "Monkey"
```
We can also do some cool things in Monkey like creating functions!
``` go
let fibonacci = fn(x) {
     if (x == 0) { 
        0 
    } else {
     if (x == 1) {
        1
        } else {
             fibonacci(x - 1) + fibonacci(x - 2);
        }
    }
};
```
Unlike python, Monkey is designed to ignore whitespace. So our fibonacci function can also be written,
``` go
letfibonacci=fn(x){if(x==0){0}else{if(x==1){1}else{fibonacci(x- 1)+fibonacci(x- 2);}}};
```
But man, that looks really gross!

Monkey even has other cool things like hashes, null values, closures, built in functions. The built in methods include len, first, last, push, and puts which simply prints the value to the console!
``` go
let x = 1
puts(x) // => 1
```

## What did this project entail?

The creation of an interpreter requires a few things,
### A parent programming language
For Monkey we used go. By having a parent programming language we can piggy back off a lot of the features in that language when creating our own interpreted language. For example, we use go's built in data types to represent our own in Monkey. These include things like integers, booleans, and strings. We wrap these data types in a struct with some additional functions for that struct to create our data type in monkey. Below is an example of how we wrap data to be used for our interpreter.
``` go
type Integer struct {
	Value int64
}

func (i *Integer) Type() ObjectType { return INTEGER_OBJ }
func (i *Integer) Inspect() string  { return fmt.Sprintf("%d", i.Value) }
```
### Lexer for lexical analysis
A lexer is used to transform the text we write into something that is more easily processed by our interpreter. What we do in the case of a lexer is we convert the text into tokens and then pass this array of tokens into or parser later. This involves identifying keywords, indentifiers for variables, and other important symbols. Once we have tokenized the entire input we can pass it the parser. Here is what our final NextToken function in our Lexer looks like.

> ## Beware it's huge!

```go
func (l *Lexer) NextToken() token.Token {
	var tok token.Token

	l.skipWhitespace()

	switch l.ch {
	case '=':
		if l.peekChar() == '=' {
			ch := l.ch
			l.readChar()
			literal := string(ch) + string(l.ch)
			tok = token.Token{Type: token.EQ, Literal: literal}
		} else {
			tok = newToken(token.ASSIGN, l.ch)
		}
	case '+':
		tok = newToken(token.PLUS, l.ch)
	case '[':
		tok = newToken(token.LBRACKET, l.ch)
	case ']':
		tok = newToken(token.RBRACKET, l.ch)
	case ':':
		tok = newToken(token.COLON, l.ch)
	case '-':
		tok = newToken(token.MINUS, l.ch)
	case '"':
		tok.Type = token.STRING
		tok.Literal = l.readString()
	case '!':
		if l.peekChar() == '=' {
			ch := l.ch
			l.readChar()
			literal := string(ch) + string(l.ch)
			tok = token.Token{Type: token.NOT_EQ, Literal: literal}
		} else {
			tok = newToken(token.BANG, l.ch)
		}
	case '/':
		tok = newToken(token.SLASH, l.ch)
	case '*':
		tok = newToken(token.ASTERISK, l.ch)
	case '<':
		tok = newToken(token.LT, l.ch)
	case '>':
		tok = newToken(token.GT, l.ch)
	case ';':
		tok = newToken(token.SEMICOLON, l.ch)
	case ',':
		tok = newToken(token.COMMA, l.ch)
	case '(':
		tok = newToken(token.LPAREN, l.ch)
	case ')':
		tok = newToken(token.RPAREN, l.ch)
	case '{':
		tok = newToken(token.LBRACE, l.ch)
	case '}':
		tok = newToken(token.RBRACE, l.ch)
	case 0:
		tok.Literal = ""
		tok.Type = token.EOF
	default:
		if isLetter(l.ch) {
			tok.Literal = l.readIdentifier()
			tok.Type = token.LookupIdent(tok.Literal)
			return tok
		} else if isDigit(l.ch) {
			tok.Type = token.INT
			tok.Literal = l.readNumber()
			return tok
		} else {
			tok = newToken(token.ILLEGAL, l.ch)
		}
	}

	l.readChar()
	return tok
}
```
### Parser for building our AST
Recursive descent parsing is a method to parse something by which we recursivley call ParseExpression while passing in the current state of our parser. We then run a simple switch to determine what should happen according to what token we have present at that point in time in our parser. The result is we traverse our array of tokens and construct a data model called an Abstract Syntax Tree w Hehich consists of a bunch of branching nodes. The AST starts from a parent node and then branches out as the parser expands on it. Our program is ready to be evaluated once we finish constructing our AST. Here is what our parseExpression function looks like!
```go
func (p *Parser) parseExpression(precedence int) ast.Expression {

	prefix := p.prefixParseFns[p.curToken.Type]

	if prefix == nil {
		p.noPrefixParseFnError(p.curToken.Type)
		return nil
	}

	leftExp := prefix()

	for !p.peekTokenIs(token.SEMICOLON) && precedence < p.peekPrecedence() {
		infix := p.infixParseFns[p.peekToken.Type]
		if infix == nil {
			return leftExp
		}
		p.nextToken()
		leftExp = infix(leftExp)
	}

	return leftExp
}
```
### Evaluator for doing maths and stuff
At first the evaluator may seem like just a simple calculator. But it is so much more. The evaluator takes our AST constructed by the parser and walks along all the branches of the tree. It does this evaluating each node and moving to the logical next node until it reaches a final statement where the final value of that tree is based on the conditions. This is the stage of our interpreter where we can make a lot of cool decisions like deciding what is true and what isnt. In Monkey we consider any value that is not null and not false to be true. This means that if we have a statment like if (1)..... the 1 will always evaluate to true.
```go
func Eval(node ast.Node, env *object.Environment) object.Object {
	switch node := node.(type) {
	// Statements
	case *ast.Program:
		return evalProgram(node, env)
	case *ast.ExpressionStatement:
		return Eval(node.Expression, env)

	// Expressions
	case *ast.IntegerLiteral:
		return &object.Integer{Value: node.Value}
	case *ast.Boolean:
		return nativeBoolToBooleanObject(node.Value)
	case *ast.InfixExpression:
		left := Eval(node.Left, env)
		right := Eval(node.Right, env)
		return evalInfixExpression(node.Operator, left, right)
	}

	return nil
}
```
