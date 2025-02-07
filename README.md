# ShipMonk PHPStan rules
Various rules we found useful in ShipMonk.
You may find some of them opinionated, so we recommend picking only those fitting your needs.

## Installation:

```sh
composer require --dev shipmonk/phpstan-rules
```

Enable all rules:
```neon
includes:
    - vendor/shipmonk/phpstan-rules/rules.neon
```

You can easily disable or reconfigure any rule. Here is a default setup used in `rules.neon` you can change:
```neon
parameters:
    shipmonkRules:
        allowComparingOnlyComparableTypes:
            enabled: true
        allowNamedArgumentOnlyInAttributes:
            enabled: true
        backedEnumGenerics:
            enabled: true
        enforceReadonlyPublicProperty:
            enabled: true
        forbidAssignmentNotMatchingVarDoc:
            enabled: true
        forbidCustomFunctions:
            enabled: true
            list: []
        forbidEnumInFunctionArguments:
            enabled: true
        forbidFetchOnMixed:
            enabled: true
        forbidMatchDefaultArmForEnums:
            enabled: true
        forbidMethodCallOnMixed:
            enabled: true
        forbidNullInBinaryOperations:
            enabled: true
            blacklist: ['===', '!==', '??']
        forbidVariableTypeOverwriting:
            enabled: true
        forbidUnsetClassField:
            enabled: true
        forbidUselessNullableReturn:
            enabled: true
        forbidUnusedException:
            enabled: true
        forbidUnusedMatchResult:
            enabled: true
        requirePreviousExceptionPass:
            enabled: true
            reportEvenIfExceptionIsNotAcceptableByRethrownOne: true
        uselessPrivatePropertyDefaultValue:
            enabled: true
        uselessPrivatePropertyNullability:
            enabled: true
```

Few rules are enabled, but do nothing unless configured, those are marked with `*`.

### AllowComparingOnlyComparableTypesRule
- Denies using comparison operators `>,<,<=,>=,<=>` over anything other than `int|string|float|DateTimeInterface`. Null is not allowed.
- Mixing different types in those operators is also forbidden, only exception is comparing floats with integers
- Mainly targets to accidental comparisons of objects, enums or arrays which is valid in PHP, but very tricky

```php
function example1(Money $fee1, Money $fee2) {
    if ($fee1 > $fee2) {} // comparing objects is denied
}

new DateTime() > '2040-01-02'; // comparing different types is denied
200 > '1e2'; // comparing different types is denied
```

### AllowNamedArgumentOnlyInAttributesRule
- Allows usage of named arguments only in native attributes
- Before native attributes, we used [DisallowNamedArguments](https://github.com/slevomat/coding-standard#slevomatcodingstandardfunctionsdisallownamedarguments). But we used Doctrine annotations, which almost "require" named arguments when converted to native attributes.
- This one is highly opinionated, you can easily disable it as described above
```php
class User {
    #[Column(type: Types::STRING, nullable: false)] // allowed
    private string $email;

    public function __construct(string $email) {
        $this->setEmail(email: $email); // forbidden
    }
}
```

### BackedEnumGenericsRule *
- Ensures that every BackedEnum child defines generic type
- This rule makes sense only when BackedEnum was hacked to be generic by stub as described in [this article](https://rnd.shipmonk.com/hacking-generics-into-backedenum-in-php-8-1/)
  - This rule does nothing if BackedEnum is not set to be generic, which is a default setup. Use following config to really start using it:
```neon
parameters:
    stubFiles:
        - BackedEnum.php.stub # see article or BackedEnumGenericsRuleTest
    ignoreErrors:
        - '#^Enum .*? has @implements tag, but does not implement any interface.$#'
```
```php
enum MyEnum: string { // missing @implements tag
    case MyCase = 'case1';
}
```

### EnforceReadonlyPublicPropertyRule
- Ensures immutability of all public properties by enforcing `readonly` modifier
- No modifier needed for readonly classes in PHP 8.2
```php
class EnforceReadonlyPublicPropertyRule {
    public int $foo; // fails, no readonly modifier
    public readonly int $bar;
}
```


### ForbidAssignmentNotMatchingVarDocRule
- Verifies if defined type in `@var` phpdoc accepts the assigned type during assignment
- No other places except assignment are checked

```php
/** @var string $foo */
$foo = $this->methodReturningInt(); // invalid var phpdoc
```

- For reasons of imperfect implementation of [type infering in phpstan-doctrine](https://github.com/phpstan/phpstan-doctrine#query-type-inference), there is an option to check only array-shapes and forget all other types by using `check-shape-only`
- This is helpful for cases where field nullability is eliminated by WHERE field IS NOT NULL which is not propagated to the inferred types
```php
/** @var array<array{id: int}> $result check-shape-only */
$result = $queryBuilder->select('t.id')
    ->from(Table::class, 't')
    ->andWhere('t.id IS NOT NULL')
    ->getResult();
```


### ForbidCustomFunctionsRule *
- Allows you to easily deny some approaches within your codebase by denying classes, methods and functions
- Configuration syntax is array where key is method name and value is reason used in error message
- Works even with interfaces, constructors and some dynamic class/method names like `$fn = 'sleep'; $fn();`
```neon
parameters:
    shipmonkRules:
        forbidCustomFunctions:
            list:
                'Namespace\SomeClass::*': 'Please use different class' # deny all methods by using * (including constructor)
                'Namespace\AnotherClass::someMethod': 'Please use anotherMethod' # deny single method
                'var_dump': 'Please remove debug code' # deny function
```
```php
new SomeClass(); // Class SomeClass is forbidden. Please use different class
(new AnotherClass())->someMethod(); // Method AnotherClass::someMethod() is forbidden. Please use anotherMethod
```

### ForbidEnumInFunctionArgumentsRule
- Guards passing native enums to native functions where it fails / produces warning or does unexpected behaviour
- Most of the array manipulation functions does not work with enums as they do implicit __toString conversion inside, but that is not possible to do with enums
- [See test](https://github.com/shipmonk-rnd/phpstan-rules/blob/master/tests/Rule/data/ForbidEnumInFunctionArgumentsRule/code.php) for all functions and their problems
```php
enum MyEnum: string {
    case MyCase = 'case1';
}

implode('', [MyEnum::MyCase]); // denied, would fail on implicit toString conversion
```


### ForbidFetchOnMixedRule
- Denies property fetch on unknown type.
- Any property fetch assumes the caller is an object with such property and therefore, the typehint/phpdoc should be fixed.
- Similar to `ForbidMethodCallOnMixedRule`
- Makes sense only on PHPStan level 8 or below, gets autodisabled on level 9
```php
function example($unknown) {
    $unknown->property; // cannot fetch property on mixed
}
```

### ForbidMatchDefaultArmForEnumsRule
- Denies using default arm in `match()` construct when native enum is passed as subject
- This rules makes sense only as a complement of [native phpstan rule](https://github.com/phpstan/phpstan-src/blob/1.7.x/src/Rules/Comparison/MatchExpressionRule.php#L94) that guards that all enum cases are handled in match arms
- As a result, you are forced to add new arm when new enum case is added. That brings up all the places in your codebase that needs new handling.
```php
match ($enum) {
    MyEnum::Case: 1;
    default: 2; // default arm forbidden
}
```

### ForbidMethodCallOnMixedRule
- Denies calling methods on unknown type.
- Any method call assumes the caller is an object with such method and therefore, the typehint/phpdoc should be fixed.
- Similar to `ForbidFetchOnMixedRule`
- Makes sense only on PHPStan level 8 or below, gets autodisabled on level 9
```php
function example($unknown) {
    $unknown->call(); // cannot call method on mixed
}
```

### ForbidNullInBinaryOperationsRule
- Denies using binary operators if null is involved on either side
- You can configure which operators are ignored in rule constructor. Default ignore is excluding only `===, !==, ??`
- Following custom setup is recommended when using latest [phpstan-strict-rules](https://github.com/phpstan/phpstan-strict-rules) and `AllowComparingOnlyComparableTypesRule` is enabled
```neon
parameters:
    shipmonkRules:
        forbidNullInBinaryOperations:
            blacklist: [
                '**', '!=', '==', '+', 'and', 'or', '&&', '||', '%', '-', '/', '*', # checked by phpstan-strict-rules
                '>', '>=', '<', '<=', '<=>', # checked by AllowComparingOnlyComparableTypesRule
                '===', '!==', '??' # valid with null involved
            ]
```
```php
function getFullName(?string $firstName, string $lastName): string {
    return $firstName . ' ' . $lastName; // denied, null involved in binary operation
}
```

### ForbidVariableTypeOverwritingRule
- Restricts variable assignment to those that does not change its type
  - Array append `$array[] = 1;` not yet supported
- Null and mixed are not taken into account, advanced phpstan types like non-empty-X are trimmed before comparison
- Rule allows type generalization and type narrowing (parent <-> child)
```php
function example(OrderId $id) {
    $id = $id->getStringValue(); // denied, type changed from object to string
}
```

### ForbidUnsetClassFieldRule
- Denies calling `unset` over class field as it causes un-initialization, see https://3v4l.org/V8uuP
- Null assignment should be used instead
```php
function example(MyClass $class) {
    unset($class->field); // denied
}
```

### ForbidUselessNullableReturnRule
- Denies marking method return type as nullable when null is never returned
- Recommended to be used together with `UselessPrivatePropertyDefaultValueRule` and `UselessPrivatePropertyNullabilityRule`
```php
class Example {
    public function example(int $foo): ?int { // null never returned
        if ($foo < 0) {
            return 0;
        }
        return $foo;
    }
}
```

### ForbidUnusedExceptionRule
- Reports forgotten exception throw (created or returned from function, but not used in any way)
```php
function validate(): void {
    new Exception(); // forgotten throw
}
```


### ForbidUnusedMatchResultRule
- Reports forgotten usage of match result
- Any `match` with at least one arm returning a value is checked
```php
match ($foo) { // unused match result
    case 'foo' => 1;
}
```


### RequirePreviousExceptionPassRule
- Detects forgotten exception pass-as-previous when re-throwing
- Checks if caught exception can be passed as argument to the call (including constructor call) in `throw` node inside the catch block
- You may encounter false-positives in some edge-cases, where you do not want to pass exception as previous, feel free to ignore those

```php
try {
    // some code
} catch (RuntimeException $e) {
    throw new LogicException('Cannot happen'); // $e not passed as previous
}
```

- If you want to be even stricter, you can set up `reportEvenIfExceptionIsNotAcceptableByRethrownOne` to `true` and the rule will start reporting even cases where the thrown exception does not have parameter matching the caught exception
  - That will force you to add the parameter to be able to pass it as previous
  - Usable only if you do not throw exceptions from libraries, which is a good practice anyway

```neon
parameters:
    shipmonkRules:
        requirePreviousExceptionPass:
            reportEvenIfExceptionIsNotAcceptableByRethrownOne: false
```
```php
class MyException extends RuntimeException {
    public function __construct() {
        parent::__construct('My error');
    }
}

try {
    // some code
} catch (RuntimeException $e) {
    throw new MyException(); // reported even though MyException cannot accept it yet
}
```

### UselessPrivatePropertyDefaultValueRule:

- Detects useless default value of a private property that is always initialized in constructor.
- Cannot handle conditions or private method calls within constructor.
- When enabled, return statements in constructors are denied to avoid false positives
- Recommended to be used with `UselessPrivatePropertyNullabilityRule` and `ForbidUselessNullableReturnRule`
```php
class Example
{
    private ?int $field = null; // useless default value

    public function __construct()
    {
        $this->field = 1;
    }
}
```

### UselessPrivatePropertyNullabilityRule:
- Detects useless nullability of a private property by checking type of all assignments.
- Recommended to be used with `UselessPrivatePropertyNullabilityRule` and `ForbidUselessNullableReturnRule` as removing useless default value may cause useless nullability to be detected
```php
class Example
{
    private ?int $field; // useless nullability

    public function __construct()
    {
        $this->field = 1;
    }

    public function setField(int $value)
    {
        $this->field = $value;
    }
}
```
