# Phase 4: Math Parser - Expression Parsing & Evaluation

## Goal
Build a robust parser that converts spoken math narration into calculated results with step-by-step explanations.

## Time Estimate: 3-4 hours

## Prerequisites
- ✅ Phase 3 complete (Whisper transcribing)
- ✅ Basic understanding of parsing concepts
- ✅ Test-driven development mindset

## Step 1: Define Domain Models (20 min)

### lib/domain/entities/math_expression.dart
```dart
// AST Node types
abstract class MathNode {
  double evaluate();
  String toInfix();
}

class NumberNode extends MathNode {
  final double value;
  
  NumberNode(this.value);
  
  @override
  double evaluate() => value;
  
  @override
  String toInfix() => value.toString();
}

class BinaryOpNode extends MathNode {
  final MathNode left;
  final MathNode right;
  final Operation operation;
  
  BinaryOpNode({
    required this.left,
    required this.right,
    required this.operation,
  });
  
  @override
  double evaluate() {
    final leftVal = left.evaluate();
    final rightVal = right.evaluate();
    
    switch (operation) {
      case Operation.add:
        return leftVal + rightVal;
      case Operation.subtract:
        return leftVal - rightVal;
      case Operation.multiply:
        return leftVal * rightVal;
      case Operation.divide:
        if (rightVal == 0) throw MathException('Division by zero');
        return leftVal / rightVal;
      case Operation.power:
        return math.pow(leftVal, rightVal).toDouble();
    }
  }
  
  @override
  String toInfix() {
    return '(${left.toInfix()} ${operation.symbol} ${right.toInfix()})';
  }
}

class UnaryOpNode extends MathNode {
  final MathNode operand;
  final UnaryOperation operation;
  
  UnaryOpNode({
    required this.operand,
    required this.operation,
  });
  
  @override
  double evaluate() {
    final val = operand.evaluate();
    
    switch (operation) {
      case UnaryOperation.negate:
        return -val;
      case UnaryOperation.sqrt:
        if (val < 0) throw MathException('Square root of negative');
        return math.sqrt(val);
      case UnaryOperation.percent:
        return val / 100;
    }
  }
  
  @override
  String toInfix() {
    switch (operation) {
      case UnaryOperation.negate:
        return '-${operand.toInfix()}';
      case UnaryOperation.sqrt:
        return '√${operand.toInfix()}';
      case UnaryOperation.percent:
        return '${operand.toInfix()}%';
    }
  }
}

enum Operation {
  add('+'),
  subtract('-'),
  multiply('×'),
  divide('÷'),
  power('^');
  
  final String symbol;
  const Operation(this.symbol);
}

enum UnaryOperation {
  negate,
  sqrt,
  percent,
}

class MathException implements Exception {
  final String message;
  MathException(this.message);
}
```

### AI Prompt for Calculation Result
```
"Create CalculationResult class that stores:
- Original transcript
- Parsed expression (AST)
- Step-by-step calculation
- Final result
- Error if any
Include helper methods for formatting"
```

## Step 2: Number Word Parser (45 min)

### lib/domain/math/number_words.dart
```dart
class NumberWordParser {
  static final Map<String, int> _ones = {
    'zero': 0, 'one': 1, 'two': 2, 'three': 3, 'four': 4,
    'five': 5, 'six': 6, 'seven': 7, 'eight': 8, 'nine': 9,
  };
  
  static final Map<String, int> _teens = {
    'ten': 10, 'eleven': 11, 'twelve': 12, 'thirteen': 13,
    'fourteen': 14, 'fifteen': 15, 'sixteen': 16,
    'seventeen': 17, 'eighteen': 18, 'nineteen': 19,
  };
  
  static final Map<String, int> _tens = {
    'twenty': 20, 'thirty': 30, 'forty': 40, 'fifty': 50,
    'sixty': 60, 'seventy': 70, 'eighty': 80, 'ninety': 90,
  };
  
  static final Map<String, int> _scales = {
    'hundred': 100,
    'thousand': 1000,
    'million': 1000000,
    'billion': 1000000000,
  };
  
  static final Map<String, double> _fractions = {
    'half': 0.5,
    'quarter': 0.25,
    'third': 1/3,
    'fourth': 0.25,
    'fifth': 0.2,
    'eighth': 0.125,
    'tenth': 0.1,
  };
  
  /// Parse number words to numeric value
  /// Examples:
  /// "twenty five" → 25
  /// "three hundred forty two" → 342  
  /// "five thousand" → 5000
  /// "two point five" → 2.5
  static double? parse(String text) {
    if (text.isEmpty) return null;
    
    // Clean and split
    final words = text.toLowerCase()
      .replaceAll('-', ' ')
      .replaceAll(',', '')
      .split(' ')
      .where((w) => w.isNotEmpty)
      .toList();
    
    // Handle decimals
    final pointIndex = words.indexOf('point');
    if (pointIndex != -1) {
      final wholePart = words.sublist(0, pointIndex);
      final decimalPart = words.sublist(pointIndex + 1);
      
      final whole = _parseInteger(wholePart) ?? 0;
      final decimal = _parseDecimal(decimalPart);
      
      return whole + decimal;
    }
    
    // Handle fractions
    for (final fraction in _fractions.keys) {
      if (words.contains(fraction)) {
        return _fractions[fraction];
      }
    }
    
    // Parse as integer
    return _parseInteger(words)?.toDouble();
  }
  
  static int? _parseInteger(List<String> words) {
    if (words.isEmpty) return null;
    
    int result = 0;
    int current = 0;
    
    for (final word in words) {
      // Check basic numbers
      if (_ones.containsKey(word)) {
        current += _ones[word]!;
      } else if (_teens.containsKey(word)) {
        current += _teens[word]!;
      } else if (_tens.containsKey(word)) {
        current += _tens[word]!;
      } else if (_scales.containsKey(word)) {
        final scale = _scales[word]!;
        if (scale >= 1000) {
          // For thousand, million, billion
          result += (current == 0 ? 1 : current) * scale;
          current = 0;
        } else {
          // For hundred
          current = (current == 0 ? 1 : current) * scale;
        }
      } else if (word == 'and') {
        // Skip connector words
        continue;
      } else {
        // Try parsing as digit
        final digit = int.tryParse(word);
        if (digit != null) {
          current = current * 10 + digit;
        }
      }
    }
    
    return result + current;
  }
  
  static double _parseDecimal(List<String> words) {
    String decimal = '0.';
    
    for (final word in words) {
      if (_ones.containsKey(word)) {
        decimal += _ones[word]!.toString();
      } else if (word == 'zero') {
        decimal += '0';
      } else {
        final digit = int.tryParse(word);
        if (digit != null) {
          decimal += digit.toString();
        }
      }
    }
    
    return double.parse(decimal);
  }
}
```

## Step 3: Expression Tokenizer (30 min)

### lib/domain/math/tokenizer.dart
```dart
enum TokenType {
  number,
  plus,
  minus,
  multiply,
  divide,
  power,
  percent,
  sqrt,
  leftParen,
  rightParen,
  equals,
  unknown,
}

class Token {
  final TokenType type;
  final String text;
  final double? value;
  
  Token({
    required this.type,
    required this.text,
    this.value,
  });
}

class Tokenizer {
  static final Map<String, TokenType> _operators = {
    'plus': TokenType.plus,
    'add': TokenType.plus,
    'and': TokenType.plus,
    'minus': TokenType.minus,
    'subtract': TokenType.minus,
    'less': TokenType.minus,
    'times': TokenType.multiply,
    'multiplied': TokenType.multiply,
    'by': TokenType.multiply,
    'divided': TokenType.divide,
    'over': TokenType.divide,
    'squared': TokenType.power,
    'cubed': TokenType.power,
    'power': TokenType.power,
    'percent': TokenType.percent,
    'square': TokenType.sqrt,
    'root': TokenType.sqrt,
    'equals': TokenType.equals,
    'is': TokenType.equals,
  };
  
  static List<Token> tokenize(String input) {
    final tokens = <Token>[];
    final words = input.toLowerCase()
      .replaceAll(RegExp(r'[^\w\s\.\-]'), ' ')
      .split(RegExp(r'\s+'))
      .where((w) => w.isNotEmpty)
      .toList();
    
    int i = 0;
    while (i < words.length) {
      final word = words[i];
      
      // Check for operators
      if (_operators.containsKey(word)) {
        final type = _operators[word]!;
        
        // Handle special cases
        if (type == TokenType.power) {
          if (word == 'squared') {
            tokens.add(Token(type: type, text: word, value: 2));
          } else if (word == 'cubed') {
            tokens.add(Token(type: type, text: word, value: 3));
          } else {
            // "to the power of" - look ahead
            if (i + 3 < words.length && 
                words[i+1] == 'to' && 
                words[i+2] == 'the' && 
                words[i+3] == 'of') {
              i += 3;
            }
            tokens.add(Token(type: type, text: 'power'));
          }
        } else if (type == TokenType.sqrt) {
          // "square root of" - look ahead
          if (word == 'square' && i + 1 < words.length && words[i+1] == 'root') {
            i++; // Skip "root"
            if (i + 1 < words.length && words[i+1] == 'of') {
              i++; // Skip "of"
            }
          }
          tokens.add(Token(type: type, text: 'sqrt'));
        } else {
          tokens.add(Token(type: type, text: word));
        }
        i++;
        continue;
      }
      
      // Check for parentheses
      if (word == 'open' || word == 'left') {
        tokens.add(Token(type: TokenType.leftParen, text: '('));
        i++;
        continue;
      }
      if (word == 'close' || word == 'right') {
        tokens.add(Token(type: TokenType.rightParen, text: ')'));
        i++;
        continue;
      }
      
      // Try to parse as number
      final numberBuffer = <String>[];
      int j = i;
      
      // Collect number words
      while (j < words.length && _isNumberWord(words[j])) {
        numberBuffer.add(words[j]);
        j++;
      }
      
      if (numberBuffer.isNotEmpty) {
        final numberText = numberBuffer.join(' ');
        final value = NumberWordParser.parse(numberText);
        
        if (value != null) {
          tokens.add(Token(
            type: TokenType.number,
            text: numberText,
            value: value,
          ));
          i = j;
          continue;
        }
      }
      
      // Skip unknown words
      i++;
    }
    
    return tokens;
  }
  
  static bool _isNumberWord(String word) {
    // Check if word could be part of a number
    return NumberWordParser.parse(word) != null ||
           word == 'point' ||
           word == 'and' ||
           word == 'hundred' ||
           word == 'thousand' ||
           word == 'million' ||
           word == 'billion';
  }
}
```

## Step 4: Expression Parser (45 min)

### lib/domain/math/parser.dart
```dart
class MathParser {
  List<Token> _tokens = [];
  int _position = 0;
  
  /// Parse transcript to AST
  MathNode? parse(String transcript) {
    try {
      _tokens = Tokenizer.tokenize(transcript);
      _position = 0;
      
      if (_tokens.isEmpty) return null;
      
      // Remove trailing "equals" tokens
      _tokens.removeWhere((t) => t.type == TokenType.equals);
      
      if (_tokens.isEmpty) return null;
      
      return _parseExpression();
    } catch (e) {
      print('Parse error: $e');
      return null;
    }
  }
  
  // Expression = Term (('+' | '-') Term)*
  MathNode _parseExpression() {
    var left = _parseTerm();
    
    while (_position < _tokens.length) {
      final token = _currentToken();
      
      if (token?.type == TokenType.plus) {
        _consume();
        final right = _parseTerm();
        left = BinaryOpNode(
          left: left,
          right: right,
          operation: Operation.add,
        );
      } else if (token?.type == TokenType.minus) {
        _consume();
        final right = _parseTerm();
        left = BinaryOpNode(
          left: left,
          right: right,
          operation: Operation.subtract,
        );
      } else {
        break;
      }
    }
    
    return left;
  }
  
  // Term = Factor (('*' | '/') Factor)*
  MathNode _parseTerm() {
    var left = _parseFactor();
    
    while (_position < _tokens.length) {
      final token = _currentToken();
      
      if (token?.type == TokenType.multiply) {
        _consume();
        final right = _parseFactor();
        left = BinaryOpNode(
          left: left,
          right: right,
          operation: Operation.multiply,
        );
      } else if (token?.type == TokenType.divide) {
        _consume();
        final right = _parseFactor();
        left = BinaryOpNode(
          left: left,
          right: right,
          operation: Operation.divide,
        );
      } else {
        break;
      }
    }
    
    return left;
  }
  
  // Factor = Power ('%')?
  MathNode _parseFactor() {
    var node = _parsePower();
    
    // Check for percent
    if (_currentToken()?.type == TokenType.percent) {
      _consume();
      node = UnaryOpNode(
        operand: node,
        operation: UnaryOperation.percent,
      );
    }
    
    return node;
  }
  
  // Power = Unary ('^' Unary)?
  MathNode _parsePower() {
    var left = _parseUnary();
    
    if (_currentToken()?.type == TokenType.power) {
      final powerToken = _consume();
      
      // Check if power has embedded value (squared/cubed)
      if (powerToken.value != null) {
        return BinaryOpNode(
          left: left,
          right: NumberNode(powerToken.value!),
          operation: Operation.power,
        );
      }
      
      // Otherwise parse the exponent
      final right = _parseUnary();
      return BinaryOpNode(
        left: left,
        right: right,
        operation: Operation.power,
      );
    }
    
    return left;
  }
  
  // Unary = ('-' | 'sqrt')? Primary
  MathNode _parseUnary() {
    final token = _currentToken();
    
    if (token?.type == TokenType.minus) {
      _consume();
      return UnaryOpNode(
        operand: _parsePrimary(),
        operation: UnaryOperation.negate,
      );
    }
    
    if (token?.type == TokenType.sqrt) {
      _consume();
      return UnaryOpNode(
        operand: _parsePrimary(),
        operation: UnaryOperation.sqrt,
      );
    }
    
    return _parsePrimary();
  }
  
  // Primary = Number | '(' Expression ')'
  MathNode _parsePrimary() {
    final token = _currentToken();
    
    if (token == null) {
      throw ParseException('Unexpected end of input');
    }
    
    if (token.type == TokenType.number) {
      _consume();
      return NumberNode(token.value!);
    }
    
    if (token.type == TokenType.leftParen) {
      _consume();
      final expr = _parseExpression();
      
      if (_currentToken()?.type != TokenType.rightParen) {
        throw ParseException('Missing closing parenthesis');
      }
      _consume();
      
      return expr;
    }
    
    throw ParseException('Expected number or parenthesis, got ${token.text}');
  }
  
  Token? _currentToken() {
    if (_position >= _tokens.length) return null;
    return _tokens[_position];
  }
  
  Token _consume() {
    final token = _tokens[_position];
    _position++;
    return token;
  }
}

class ParseException implements Exception {
  final String message;
  ParseException(this.message);
}
```

## Step 5: Calculation Service (30 min)

### lib/application/math/calculation_service.dart
```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';

class CalculationService {
  final MathParser _parser = MathParser();
  
  CalculationResult calculate(String transcript) {
    try {
      // Parse to AST
      final ast = _parser.parse(transcript);
      
      if (ast == null) {
        return CalculationResult.error(
          transcript: transcript,
          error: 'Could not parse expression',
        );
      }
      
      // Evaluate
      final result = ast.evaluate();
      
      // Generate steps
      final steps = _generateSteps(ast, result);
      
      return CalculationResult(
        transcript: transcript,
        expression: ast.toInfix(),
        result: result,
        steps: steps,
      );
    } catch (e) {
      return CalculationResult.error(
        transcript: transcript,
        error: e.toString(),
      );
    }
  }
  
  List<CalculationStep> _generateSteps(MathNode ast, double result) {
    final steps = <CalculationStep>[];
    
    // Show original expression
    steps.add(CalculationStep(
      description: 'Original',
      expression: ast.toInfix(),
    ));
    
    // For complex expressions, show intermediate steps
    if (ast is BinaryOpNode) {
      final leftVal = ast.left.evaluate();
      final rightVal = ast.right.evaluate();
      
      steps.add(CalculationStep(
        description: 'Calculate ${ast.operation.symbol}',
        expression: '$leftVal ${ast.operation.symbol} $rightVal = $result',
      ));
    }
    
    // Final result
    steps.add(CalculationStep(
      description: 'Result',
      expression: result.toString(),
    ));
    
    return steps;
  }
}

class CalculationResult {
  final String transcript;
  final String? expression;
  final double? result;
  final List<CalculationStep> steps;
  final String? error;
  
  CalculationResult({
    required this.transcript,
    this.expression,
    this.result,
    this.steps = const [],
    this.error,
  });
  
  factory CalculationResult.error({
    required String transcript,
    required String error,
  }) {
    return CalculationResult(
      transcript: transcript,
      error: error,
    );
  }
  
  bool get hasError => error != null;
  
  String get displayResult {
    if (hasError) return 'Error: $error';
    if (result == null) return 'No result';
    
    // Format nicely
    if (result! == result!.roundToDouble()) {
      return result!.toInt().toString();
    }
    return result!.toStringAsFixed(2);
  }
}

class CalculationStep {
  final String description;
  final String expression;
  
  CalculationStep({
    required this.description,
    required this.expression,
  });
}
```

## Step 6: Write Comprehensive Tests (45 min)

### test/domain/math/parser_test.dart
```dart
import 'package:flutter_test/flutter_test.dart';

void main() {
  group('NumberWordParser', () {
    test('parses basic numbers', () {
      expect(NumberWordParser.parse('five'), 5);
      expect(NumberWordParser.parse('twenty'), 20);
      expect(NumberWordParser.parse('hundred'), 100);
    });
    
    test('parses compound numbers', () {
      expect(NumberWordParser.parse('twenty five'), 25);
      expect(NumberWordParser.parse('ninety nine'), 99);
      expect(NumberWordParser.parse('forty two'), 42);
    });
    
    test('parses large numbers', () {
      expect(NumberWordParser.parse('five hundred'), 500);
      expect(NumberWordParser.parse('three thousand'), 3000);
      expect(NumberWordParser.parse('twenty five thousand'), 25000);
      expect(NumberWordParser.parse('five million'), 5000000);
    });
    
    test('parses complex numbers', () {
      expect(NumberWordParser.parse('five thousand three hundred twenty one'), 5321);
      expect(NumberWordParser.parse('one million two hundred thousand'), 1200000);
    });
    
    test('parses decimals', () {
      expect(NumberWordParser.parse('five point seven'), 5.7);
      expect(NumberWordParser.parse('zero point nine'), 0.9);
      expect(NumberWordParser.parse('three point one four'), 3.14);
    });
  });
  
  group('MathParser', () {
    final parser = MathParser();
    
    test('parses basic operations', () {
      expect(parser.parse('five plus three')?.evaluate(), 8);
      expect(parser.parse('ten minus four')?.evaluate(), 6);
      expect(parser.parse('six times seven')?.evaluate(), 42);
      expect(parser.parse('eight divided by two')?.evaluate(), 4);
    });
    
    test('parses complex expressions', () {
      expect(parser.parse('twenty five times four plus ten')?.evaluate(), 110);
      expect(parser.parse('hundred divided by four minus five')?.evaluate(), 20);
    });
    
    test('handles order of operations', () {
      expect(parser.parse('two plus three times four')?.evaluate(), 14);
      expect(parser.parse('ten minus two times three')?.evaluate(), 4);
    });
    
    test('parses percentages', () {
      expect(parser.parse('ten percent')?.evaluate(), 0.1);
      expect(parser.parse('fifty percent')?.evaluate(), 0.5);
      expect(parser.parse('twenty five percent')?.evaluate(), 0.25);
    });
    
    test('parses powers', () {
      expect(parser.parse('five squared')?.evaluate(), 25);
      expect(parser.parse('two cubed')?.evaluate(), 8);
      expect(parser.parse('three to the power of four')?.evaluate(), 81);
    });
    
    test('parses square roots', () {
      expect(parser.parse('square root of sixteen')?.evaluate(), 4);
      expect(parser.parse('square root of twenty five')?.evaluate(), 5);
    });
    
    test('handles parentheses', () {
      final expr = 'open parenthesis five plus three close parenthesis times two';
      expect(parser.parse(expr)?.evaluate(), 16);
    });
    
    test('handles "equals" in transcript', () {
      expect(parser.parse('five plus three equals')?.evaluate(), 8);
      expect(parser.parse('two times four is')?.evaluate(), 8);
    });
  });
}
```

### AI Prompt for More Tests
```
"Write 20 more test cases for the math parser covering:
- Edge cases (division by zero, negative numbers)
- Mixed operations
- Common speech patterns
- Error scenarios
Include expected results and explanations"
```

## Step 7: Integration with UI (20 min)

### Update record_screen.dart
```dart
// Add to the widget
Widget _buildResultCard(CalculationResult? result) {
  if (result == null) return SizedBox.shrink();
  
  return Container(
    margin: EdgeInsets.all(20),
    padding: EdgeInsets.all(16),
    decoration: BoxDecoration(
      color: result.hasError ? Colors.red.shade50 : Colors.green.shade50,
      borderRadius: BorderRadius.circular(16),
      border: Border.all(
        color: result.hasError ? Colors.red : Colors.green,
        width: 2,
      ),
    ),
    child: Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        Text(
          'Result',
          style: Theme.of(context).textTheme.titleMedium,
        ),
        SizedBox(height: 8),
        Text(
          result.displayResult,
          style: Theme.of(context).textTheme.headlineMedium?.copyWith(
            color: result.hasError ? Colors.red : Colors.green.shade700,
          ),
        ),
        if (!result.hasError && result.steps.isNotEmpty) ...[
          SizedBox(height: 16),
          Text(
            'Steps:',
            style: Theme.of(context).textTheme.labelLarge,
          ),
          ...result.steps.map((step) => Padding(
            padding: EdgeInsets.only(top: 4),
            child: Text(
              '${step.description}: ${step.expression}',
              style: Theme.of(context).textTheme.bodyMedium,
            ),
          )),
        ],
      ],
    ),
  );
}
```

### Wire up the provider
```dart
@riverpod
class CalculationNotifier extends _$CalculationNotifier {
  late final CalculationService _service;
  
  @override
  CalculationResult? build() {
    _service = CalculationService();
    
    // Listen to transcript updates
    ref.listen(transcriptProvider, (previous, transcript) {
      if (transcript != previous && transcript.isNotEmpty) {
        calculate(transcript);
      }
    });
    
    return null;
  }
  
  void calculate(String transcript) {
    state = _service.calculate(transcript);
  }
}
```

## Testing Checkpoints

### ✅ Checkpoint 1: Number Parsing
```dart
// Test in debug console
NumberWordParser.parse('twenty five'); // Should return 25
NumberWordParser.parse('three point five'); // Should return 3.5
```

### ✅ Checkpoint 2: Tokenization
```dart
Tokenizer.tokenize('five plus three');
// Should return tokens: [NUMBER(5), PLUS, NUMBER(3)]
```

### ✅ Checkpoint 3: AST Building
```dart
final ast = MathParser().parse('five plus three');
print(ast?.toInfix()); // Should print "(5 + 3)"
```

### ✅ Checkpoint 4: Evaluation
```dart
final result = MathParser().parse('twenty five times four')?.evaluate();
print(result); // Should print 100
```

### ✅ Checkpoint 5: End-to-End
```dart
// Speak "twenty five times four plus ten percent"
// Should see result: 110
```

## Common Issues & Solutions

### Issue: Numbers not parsing correctly
**Solution**: Check word spacing and hyphenation in tokenizer

### Issue: Wrong operation order
**Solution**: Verify precedence in parser (multiply/divide before add/subtract)

### Issue: Percent calculation wrong
**Solution**: Ensure percent is 
treated as "of previous value" in context

### Issue: Parser throws exception
**Solution**: Add better error recovery and skip unknown tokens

## Performance Optimization

1. **Cache parsed numbers**: Store frequently used number words
2. **Optimize tokenizer**: Use regex for faster splitting
3. **Lazy evaluation**: Only calculate when needed
4. **Memoization**: Cache AST for repeated expressions

## Success Criteria

✅ 40+ test cases passing
✅ Common expressions parse correctly
✅ Proper order of operations
✅ Handles speech variations
✅ Clear error messages
✅ Step-by-step results displayed

## Next Steps

With core functionality complete:
1. Add more operations (factorial, log, trig)
2. Support variables ("let x equal five")
3. Add undo/redo for calculations
4. Export calculation history
5. Add voice feedback for results

## Quick Test Expressions

Test these manually:
```
"five plus three" → 8
"twenty minus seven" → 13
"six times nine" → 54
"hundred divided by four" → 25
"two plus three times four" → 14
"fifty percent of eighty" → 40
"five squared" → 25
"square root of sixteen" → 4
"twenty five times four plus ten percent" → 110
```

## AI Prompt for Enhancement

```
"Enhance math parser with:
- Support for 'of' operator (50% of 100)
- Memory functions (store, recall)
- Constants (pi, e)
- Better error messages
Show implementation with tests"
```

Remember: Start simple, test thoroughly, iterate!