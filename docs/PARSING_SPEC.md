# Mental Math Parsing Specification

## Overview

This document defines the grammar and parsing rules for converting spoken mathematical narration into computable expressions.

## Supported Operations

### Basic Arithmetic
- Addition: "plus", "add", "and"
- Subtraction: "minus", "subtract", "less", "take away"
- Multiplication: "times", "multiplied by", "by"
- Division: "divided by", "over", "split by"

### Advanced Operations
- Percentage: "percent", "percent of"
- Powers: "squared", "cubed", "to the power of", "raised to"
- Roots: "square root of", "cube root of"
- Parentheses: "open parenthesis", "close parenthesis"

## Number Recognition

### Cardinal Numbers
```
"zero" → 0
"one" → 1
"two" → 2
...
"nine" → 9
"ten" → 10
"eleven" → 11
"twelve" → 12
...
"twenty" → 20
"thirty" → 30
...
"ninety" → 90
```

### Compound Numbers
```
"twenty-one" → 21
"forty-five" → 45
"ninety-nine" → 99
```

### Large Numbers
```
"hundred" → ×100
"thousand" → ×1000
"million" → ×1000000
"billion" → ×1000000000
```

### Complex Examples
```
"five hundred" → 500
"three thousand" → 3000
"twenty-five hundred" → 2500
"five thousand three hundred twenty-one" → 5321
"two million five hundred thousand" → 2500000
```

### Decimals
```
"point" → .
"five point seven" → 5.7
"zero point nine" → 0.9
"three point one four" → 3.14
"twelve point zero five" → 12.05
```

### Fractions
```
"half" → 0.5
"quarter" → 0.25
"three quarters" → 0.75
"one third" → 0.333...
"two thirds" → 0.666...
"one eighth" → 0.125
```

## Grammar Rules (BNF-like)

```bnf
expression     ::= term ((ADD | SUBTRACT) term)*
term           ::= factor ((MULTIPLY | DIVIDE) factor)*
factor         ::= power (PERCENT)?
power          ::= unary (POWER unary)?
unary          ::= (SQRT | NEGATE)? primary
primary        ::= NUMBER | LPAREN expression RPAREN

NUMBER         ::= cardinal | decimal | fraction | compound
ADD            ::= "plus" | "add" | "and"
SUBTRACT       ::= "minus" | "subtract" | "less" | "take away"
MULTIPLY       ::= "times" | "multiplied by" | "by"
DIVIDE         ::= "divided by" | "over" | "split by"
PERCENT        ::= "percent"
POWER          ::= "squared" | "cubed" | "to the power of"
SQRT           ::= "square root of" | "root of"
NEGATE         ::= "negative" | "minus"
LPAREN         ::= "open parenthesis" | "left paren"
RPAREN         ::= "close parenthesis" | "right paren"
```

## Parsing Examples

### Simple Arithmetic
| Input | Parse Tree | Result |
|-------|------------|--------|
| "five plus three" | Add(5, 3) | 8 |
| "ten minus four" | Subtract(10, 4) | 6 |
| "six times seven" | Multiply(6, 7) | 42 |
| "eight divided by two" | Divide(8, 2) | 4 |

### Complex Expressions
| Input | Parse Tree | Result |
|-------|------------|--------|
| "twenty-five times four plus ten" | Add(Multiply(25, 4), 10) | 110 |
| "one hundred divided by four minus five" | Subtract(Divide(100, 4), 5) | 20 |
| "three plus four times five" | Add(3, Multiply(4, 5)) | 23 |

### Percentages
| Input | Parse Tree | Result |
|-------|------------|--------|
| "ten percent" | Percent(10) | 0.1 |
| "fifty percent of eighty" | Multiply(Percent(50), 80) | 40 |
| "twenty plus ten percent" | Add(20, Multiply(20, Percent(10))) | 22 |
| "hundred minus twenty-five percent" | Subtract(100, Multiply(100, Percent(25))) | 75 |

### Powers and Roots
| Input | Parse Tree | Result |
|-------|------------|--------|
| "five squared" | Power(5, 2) | 25 |
| "two cubed" | Power(2, 3) | 8 |
| "three to the power of four" | Power(3, 4) | 81 |
| "square root of sixteen" | Sqrt(16) | 4 |
| "square root of nine plus sixteen" | Add(Sqrt(9), 16) | 19 |

### Parentheses
| Input | Parse Tree | Result |
|-------|------------|--------|
| "open parenthesis five plus three close parenthesis times two" | Multiply(Group(Add(5, 3)), 2) | 16 |
| "ten times open parenthesis four minus two close parenthesis" | Multiply(10, Group(Subtract(4, 2))) | 20 |

### Mental Math Patterns
| Input | Parse Tree | Result |
|-------|------------|--------|
| "twenty-five times four is one hundred" | Multiply(25, 4) | 100 |
| "carry the one" | (context-dependent) | - |
| "add two zeros" | Multiply(prev, 100) | - |
| "move the decimal point" | Multiply/Divide by 10^n | - |

## Edge Cases

### Ambiguity Resolution
1. **"one hundred and five"** → 105 (not 100 + 5)
2. **"twenty one thousand"** → 21,000 (not 20 + 1000)
3. **"five and a half"** → 5.5 (not 5 + 0.5)

### Error Recovery
1. **Incomplete expressions**: "five plus" → Prompt for completion
2. **Invalid operations**: "five divided by zero" → Error with explanation
3. **Unrecognized words**: Skip or request clarification

### Context Handling
```
"twenty-five"          [Store: 25]
"times four"           [Apply: 25 × 4 = 100]
"plus ten percent"     [Apply: 100 + (100 × 0.1) = 110]
```

## Tokenization Rules

### Token Priority
1. Number words (highest)
2. Operation keywords
3. Modifiers (percent, squared)
4. Grouping (parentheses)
5. Filler words (ignored)

### Ignored Words
- "um", "uh", "er"
- "equals", "is", "gives"
- "the", "a", "an"
- "please", "calculate"

## Implementation Notes

### Parser State Machine
```
START → NUMBER → OPERATOR → NUMBER → [OPERATOR|END]
         ↓                     ↑
    PARENTHESIS → EXPRESSION →┘
```

### AST Node Types
```dart
abstract class MathNode {}

class NumberNode extends MathNode {
  final double value;
}

class BinaryOpNode extends MathNode {
  final MathNode left;
  final MathNode right;
  final Operation op;
}

class UnaryOpNode extends MathNode {
  final MathNode operand;
  final UnaryOp op;
}
```

### Evaluation Strategy
1. Build complete AST
2. Validate tree structure
3. Evaluate recursively from leaves
4. Apply operator precedence
5. Handle precision with decimal arithmetic

## Test Coverage Requirements

### Categories
1. **Basic Operations** (10 tests)
2. **Number Recognition** (10 tests)
3. **Complex Expressions** (10 tests)
4. **Percentages** (5 tests)
5. **Powers/Roots** (5 tests)
6. **Parentheses** (5 tests)
7. **Error Cases** (5 tests)

### Example Test Cases
```dart
test('simple addition', () {
  expect(parse("five plus three"), equals(8));
});

test('compound numbers', () {
  expect(parse("twenty-five"), equals(25));
});

test('order of operations', () {
  expect(parse("two plus three times four"), equals(14));
});

test('percentage calculation', () {
  expect(parse("fifty percent of eighty"), equals(40));
});

test('nested parentheses', () {
  expect(parse("open parenthesis two plus three close parenthesis squared"), 
         equals(25));
});
```

## Performance Targets

- Parse time: < 10ms for typical expression
- Memory: < 1KB per expression
- Accuracy: 100% for well-formed input
- Error recovery: 95% success rate

## Future Enhancements

1. **Variables**: "let x equal five"
2. **Functions**: "sine of thirty degrees"
3. **Units**: "five meters plus three feet"
4. **Comparisons**: "is five greater than three"
5. **Statistics**: "average of five, ten, fifteen"