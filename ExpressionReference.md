# Expression reference #
This wiki page contains a list of valid Supersonic expression types with short usage descriptions and links to implementation.

# General notes #
Expressions are units of computation carried out on row level, that is they operate on column values of a single row, although in some cases it is possible to pass simple state between rows. For information on operations, which are higher level computations spanning multiple rows, check out [OperationReference](OperationReference.md).

# Expressions #

## Terminal (leaf-node) expressions ##
Terminal expressions have no sub-expressions - they are at the leaf level of an expression tree. Their type is always predetermined and the values resemble their programmatic argument. DataType is a meta-type - it takes a plain type as a value.

  * `ConstInt32(int32 value)`
  * `ConstInt64(int64 value)`
  * `ConstUint32(uint32 value)`
  * `ConstUint64(uint64 value)`
  * `ConstFloat(float value)`
  * `ConstDouble(double value)`
  * `ConstBool(bool value)`
  * `ConstDate(int32 value)` - the value is days since epoch.
  * `ConstDateTime(int64 value)` - the value is microseconds since epoch.
  * `ConstString(StringPiece value)`
  * `ConstBinary(StringPiece value)`
  * `ConstDataType(DataType value)`
  * `template <DataType type> TypedConst(value)` - templated wrapper for the above
  * `Sequence()` - an INT64 expression, when evaluated produces consecutive integers, starting at zero, consistent across multiple calls to evaluate.
  * `RandInt32(RandomBase* random_generator)` - produces a sequence of pseudo-random INT32 numbers, as generated by the random generator (defined in utils/random\_base.h (link!)). Does not take ownership of random\_generator.
  * `RandInt32() - a shortcut for RandInt32(MTRandom())`
  * `Null(DataType type)` - creates a Null expression of the given `type`.

## Projecting expressions ##
Projecting expressions are expressions that take some inputs (either the input view, or a number of other expressions), and output their projection. A projection can, in particular, restrict the number of columns (to, e.g., one, for further processing by expressions expecting a single-column input), rename the columns (using an aliasing projector), concatenate a number of input expressions into a single multi-column expression, and more. All this is performed without copying the data, so projecting expressions are cheap in execution. They do not change column types.
For the most common uses (selecting a single column from the input, concatenating a number of expressions into a single expression) there are convenience shortcuts available. For other uses the caller has to supply a projector, see the declarations in projector.h (link!) for defining projectors.

Expressions that project columns from the input:
  * `NamedAttribute(string name)` - returns a single-column expression that selects the column with the corresponding name from the input (binding fails if no such column exists).
  * `AttributeAt(size_t position)` - returns a single-column expression that selects the column at the given position from the input (binding fails if position is out of bounds)
  * `InputAttributeProjection(SingleSourceProjector* projector)` - a generalization of both of the above, can return multiple columns. See projector.h (link!) for defining projectors. Takes ownership of projector.

Expressions that project columns from the outputs of other expressions:
  * `Flat(ExpressionList* inputs)` - returns a single expression concatenating the input expressions. Takes ownership of inputs.
  * `Projection(ExpressionList* sources, MultiSourceProjector* projector)` - a generalization of the above, selects an arbitrary subset of the columns of the input expressions. See projector.h for defining projectors. Takes ownership of sources and projector.

## Arithmetic expressions ##
The inputs of arithmetic expressions always have to be of one of the following types:
  * INT32
  * UINT32
  * INT64
  * UINT64
  * FLOAT
  * DOUBLE
The result of the expression will have the "smallest common containing type" of its inputs, which is defined as follows:
  1. The smallest common containing type of T1 and T2 is integer if and only if T1 and T2 are both integer, otherwise it is a floating-point type.
  1. The smallest common containing type of T1 and T2 is small (INT32, UINT32 or FLOAT) if and only if both T1 and T2 are small, otherwise it's large (INT64, UINT64, DOUBLE).
  1. The smallest common containing type of T1 and T2 is an unsigned integer if and only if both T1 and T2 are unsigned integers.

These three rules define the smallest common containing type uniquely.
Some of the arithmetic expressions (ones that have the "division by zero" problem) come with differing policies - Nulling, Signaling or (possibly) Quiet. See policy notes (ADD LINK) for details on this.
All expressions return NULL if any of the inputs is NULL.

  * `Plus(Expression* a, Expression* b)` - returns the sum of a and b. The result type is the smallest common containing type.
  * `Minus(Expression* a, Expression* b)` - returns the difference of a and b. The result type is the smallest common containing type (in particular UINT32 - UINT32 is UINT32).
  * `Multiply(Expression* a, Expression* b)` - returns the product of a and b. The result type is the smallest common containing type.
  * `DivideSignaling(Expression* a, Expression* b)` - returns the quotient of a and b. The result type is always DOUBLE. Evaluation fails on division by zero (see policy notes). Note that NULL / 0 evaluates to NULL (and not a failure).
  * `DivideNulling(Expression* a, Expression* b)` - returns the quotient of a and b. The result type is always DOUBLE. Evaluation returns NULL on division by zero (see policy notes).
  * `DivideQuiet(Expression* a, Expression* b)` - returns the quotient of a and b. The result type is always DOUBLE. Evaluation returns NaN on division by zero (see policy notes). Note that NULL / 0 evaluates to NULL (and not a NaN).
  * `CppDivideSignaling(Expression* a, Expression* b)` - returns the quotient of a and b. The result type is the smallest common containing type, in particular if both the inputs are integers, this performs a (C++ style) integer division. Evaluation fails on division by zero (see policy notes). Note that NULL / 0 evaluates to NULL (and not a failure).
  * `CppDivideNulling(Expression* a, Expression* b)` - returns the quotient of a and b. The result type is the smallest common containing type, in particular if both the inputs are integers, this performs a (C++ style) integer division. Evaluation returns NULL on division by zero (see policy notes).
  * `Negate(Expression* a)` - returns the negation of a (that is, -a). The result type is the input type, with the exception that for unsigned types (UINT32 and UINT64) the result type is the corresponding signed type.
  * `ModulusSignaling(Expression* a, Expression* b)` - returns the remainder of dividing a by b (that is, a % b). The result type is the smallest common containing type. Both the inputs have to be integers. Evaluation fails if b is zero (see policy notes). Note that NULL % 0 evaluates to NULL (and not a failure).
  * `ModulusNulling(Expression* a, Expression* b)` - returns the remainder of dividing a by b (that is, a % b). The result type is the smallest common containing type. Both the inputs have to be integers. Evaluation returns NULL if b is zero (see policy notes).


## Comparison expressions ##
Below are expressions that return a boolean value after checking whether a comparison relations holds for their inputs. NULL will always be returned if one of the inputs is NULL.

  * `Equal(Expression* a, Expression* b)` - returns the semantic equivalent of the a == b test. If the expressions are of a different type, but can be reconciled, the reconciliation is performed.
  * `NotEqual(Expression* a, Expression* b)` - returns the semantic equivalent of the a != b test. If the expressions are of a different type, but can be reconciled, the reconciliation is performed.
  * `Less(Expression* a, Expression* b)`
  * `LessOrEqual(Expression* a, Expression* b)`
  * `Greater(Expression* a, Expression* b)`
  * `GreaterOrEqual(Expression* a, Expression* b)` - returns the semantic equivalent of the appropriate (respectively `a < b`, `a <= b`, `a > b`, `a >= b`) test. If the expressions are of a different type, but can be reconciled, the reconciliation is performed. Note that this is different from the C++ semantics, where int(-1) < uint(0) returns false. For strings and binaries this is a lexicographic comparison. For DataTypes this is a comparison based on the textual representation of the datatypes.
  * `IsOdd(Expression* a)`
  * `IsEven(Expression* a)` - checks whether the input is odd (even). Requires the input to be an integer.
  * `InInt32(Expression* a, vector<int32> values)` -  and similarly `InInt64`, `InUint32`, `InUint64`, `InFloat`, `InDouble`, `InBool`, `InDate`, `InDateTime`, `InString`, `InBinary` - checks whether a is one of the elements listed in values. The argument a is forcibly cast to the type specified during creation, note that this may cause precision loss and/or wrapping (in case of INT64->INT32 and signed `<`-`>` unsigned casts). Returns NULL if a is NULL.


## Date and Date Time expressions ##
Date and Date Time expressions operate on Dates (days from epoch) and Date Times (microseconds from epoch). They always return NULL if any of the inputs is NULL.

  * `ConstDateTime(const StringPiece& value)` - returns a Supersonic DATETIME constant, from the string format YYYY/MM/DD-HH:MM:SS. Returns a NULL constant if value is out of bounds or has invalid format.
  * `ConstDateTimeFromMicrosecondsSinceEpoch(int64 value)` - returns a Supersonic DATETIME constant corresponding to value microseconds after epoch (01/01/1970-00:00:00.0).
  * `ConstDateTimeFromSecondsSinceEpoch(double value)` - returns a Supersonic DATETIME constant corresponding to value seconds after epoch. Note that value is a double.
  * `Now()` - returns a Supersonic DATETIME constant which corresponds to the UTC time of creating the expression (this is creation time, not evaluation time).
  * `UnixTimestamp(Expression* datetime)` - returns an INT64 timestamp (seconds from epoch) created from the input DATETIME.
  * `FromUnixTime(Expression* timestamp)` - returns a DATETIME created from the input timestamp (INT64 seconds from epoch)
  * `MakeDate(Expression* year, Expression* month, Expression* day)` - returns a DATE corresponding to 00:00:00 of the date described by the inputs. The inputs are expected to be integers, 1970 upwards for year, 1..12 for month and 1..31 for day.
  * `MakeDateTime(Expression* year, Expression* month, Expression* day, Expression* hour, Expression* minute, Expression* second)` - returns a DATETIME described by the inputs. The inputs are expected to be integers, 1970 upwards for year, 1..12 for month, 1..31 for day, 0..24 for hour and 0..59 for minute and second.
  * `ParseDateTime(StringPiece format, Expression* input)` - parse a Supersonic STRING input to a DATETIME according to the format string. The format is described as in man strptime. Whitespace at either end of parsed string is accepted. Out of range, unparseable and badly formatted strings are converted to NULL.
  * `Year(Expression* time)` - return the year of the date described by the DATETIME or DATE argument. The year is returned as an INT32. The argument is treated as a UTC time.
  * `Quarter(Expression* time)` - return the quarter of the year in which the the date described by the DATETIME or DATE argument lies. The quarter is returned as an INT32 in the 1..4 range. The argument is treated as a UTC time.
  * `Month(Expression* time)` - return the month of the year in which the the date described by the DATETIME or DATE argument lies. The month is returned as an INT32 in the 1..12 range. The argument is treated as a UTC time.
  * `Day(Expression* time)` - return the day of month in which the the date described by the DATETIME or DATE argument lies. The day of month is returned as an INT32 in the 1..31 range. The argument is treated as a UTC time.
  * `Weekday(Expression* time)` - return the day of the week in which the the date described by the DATETIME or DATE argument lies. The day of the week is returned as an INT32 in the 0..6 range, where 0 denoted Monday and 6 denotes Sunday. The argument is treated as a UTC time.
  * `YearDay(Expression* time)` - return the day of the year in which the the date described by the DATETIME or DATE argument lies. The day is returned as an INT32 in the 1..366 range. The argument is treated as a UTC time.
  * `Hour(Expression* time)` - return the hour of the day in which the the date described by the DATETIME argument lies (note - this does not support DATE arguments, they would make no sense - use a ConstInt32(0) instead). The hour is returned as an INT32 in the 0..23 range. The argument is treated as a UTC time.
  * `Minute(Expression* time)` - return the minute of the hour in which the the date described by the DATETIME argument lies. The minute is returned as an INT32 in the 0..59 range. The argument is treated as a UTC time.
  * `Second(Expression* time)` - return the second of the minute in which the date described by the DATETIME argument lies. The second is returned as an INT32 in the 0..59 range.
  * `Microsecond(Expression* time)` - return the microsecond of the second in which the date described by the DATETIME argument lies. The microsecond is returned as an INT32 in the 0..999999 range.
  * `AddMinute(Expression* time)` - returns the DATETIME describing a moment one minute later. time can be an DATETIME or DATE (interpreted as 00:00:00 of that day).
  * `AddMinutes(Expression* time, Expression* number_of_minutes)` - returns the DATETIME number\_of\_minutes minutes later than time. number\_of\_minutes is expected to be an integer, time can be an DATETIME or DATE (interpreted as 00:00:00 of that day).
  * `AddDay(Expression* time)` - returns the DATETIME one day later (in UTC) than time.
  * `AddDays(Expression* time, Expression* number_of_days)` - returns the DATETIME number\_of\_days days later than time (in UTC). number\_of\_days is expected to be an integer, time can be an DATETIME or DATE (interpreted as 00:00:00 of that day).
  * `AddMonth(Expression* time)` - returns the DATETIME one month later than time (in UTC). Note that this is less efficient than the previous two functions, as it requires the taking into account of month lengths, leap years, etc. time can be an DATETIME or DATE (interpreted as 00:00:00 of that day).
  * `AddMonths(Expression* time, Expression* number_of_months)` - returns the DATETIME number\_of\_months later than time (in UTC). number\_of\_months is expected to be an integer, time can be an DATETIME or DATE (interpreted as 00:00:00 of that day).


## Logical expressions ##
Logical expressions operate on boolean inputs and outputs. They use ternary logic, with true, false and NULL values, where for instance NULL OR TRUE is true. They also use short-circuiting, so if the left-hand-side is enough to get the value of the whole expression, the right-hand-side will not be calculated - this is a bit of a simplification, but also a good first approximation (note that this does not apply to XOR). The inputs always have to be booleans (and the output is also boolean).
  * `And(Expression* a, Expression* b)` - the logical conjunction of the arguments.
  * `Or(Expression* a, Expression* b)` - the logical alternative of the arguments.
  * `AndNot(Expression* a, Expression* b)` - this (somewhat counterintuitively) means (NOT a) AND b.
  * `Xor(Expression* a, Expression* b)` - the exclusive alternative of the arguments.
  * `Not(Expression* a)` - the negation of the argument.


## Conversions ##
Expressions that serve to change the type of the data. Will always return NULL for NULL input.

  * `CastTo(DataType to_type, Expression* source)` - casts source to to\_type. The casts allowed by Supersonic are as follows: between all integer types, from any numeric type to any floating point type, from STRING to BINARY and the opposite, and from DATE to DATETIME (and also from any type to itself). Other casts will fail at binding time.
  * `ParseStringQuiet(DataType to_type, Expression* source)` - parses the source STRING to the specified datatype. Will produce garbage outputs on ill-formatted inputs, use with care - if not 100% sure, use ParseStringNulling instead. Boolean parsing accepts "true", "yes", "false" and "no" inputs. DATEs are parsed in the YYYY/MM/DD format, DATETIMEs in YYYY/MM/DD-HH:MM:SS. DATATYPEs expect their textual representation. STRINGs and BINARYs cannot be parsed. Returns NULL if the input is NULL. See policy notes for more on Quiet expressions.
  * `ParseStringNulling(DataType to_type, Expression* source)` - the same semantics as the above function, except that it returns NULL if parsing failed (because the string was malformed or the result out of range). See policy notes for more on Nulling expressions.


## Control flow expressions ##
  * `If(Expression* condition, Expression* then, Expression* otherwise)` - condition has to be a BOOL expression, while the types of then and otherwise have to be reconcilable, and the result is their smallest common containing type (see arithmetic expressions for type reconciliation). For those inputs for which condition evaluates to true, then is returned, otherwise - otherwise is returned.
  * `NullingIf(Expression* condition, Expression* then, Expression* otherwise)` - condition has to be a BOOL expression, while the types of then and otherwise have to be reconcilable (see arithmetic expressions for type reconciliation). For those inputs for which condition evaluates to true, then is returned, if it evaluates to false - otherwise is returned, if it evaluates to NULL - NULL is returned.
  * `IsNull(Expression* e)` - returns true if e is NULL, false otherwise. Never returns NULL. The return type is the same as the type of e.
  * `IfNull(Expression* e, Expression* substitute)` - returns e if e is not NULL, substitute if e is NULL. e and substitute types have to be reconcilable, and the result is their smallest common containing type (see arithmetic expressions for type reconciliation). Note that the result will be NULL if both e and substitute are NULL.
  * `Case(ExpressionList* arguments)` - evaluates the first argument (the switch). Then compares the switch to arguments on positions three, five, and so on, until it finds a match. If the switch matches argument 2k+1, then argument 2k+2 is returned. If no match occurs (in particular no match occurs if the switch is NULL), the second argument is returned. The number of arguments has to be even, the odd-placed arguments have to be reconcilable and the even-placed arguments have to be reconcilable. The result type is the smallest common containing type of the even-placed arguments.


## Mathematical expressions ##
All of them return NULL if any of the inputs is NULL. Unless noted otherwise, the return type is DOUBLE, while the input type can be any numeric type. All floating point calculations follow the IEEE 754 standard, unless noted otherwise.

  * `Exp(Expression* argument)` - the exponent (e to the power `argument`)
  * `LnNulling(Expression* argument)` - the natural logarithm of the `argument`. If the result would be infinity or NaN, returns NULL instead. See policy notes for more on Nulling expressions.
  * `LnQuiet(Expression* argument)` - the natural logarithm of the `argument`. See policy notes for more on Quiet expressions.
  * `Log10Nulling(Expression* argument)` - the base 10 logarithm of the `argument`. If the result would be infinity or NaN, returns NULL instead. See policy notes for more on Nulling expressions.
  * `Log10Quiet(Expression* argument)` - the base 10 logarithm of the `argument`. See policy notes for more on Quiet expressions.
  * `Log2Nulling(Expression* argument)` - the base 2 logarithm of the `argument`. If the result would be infinity or NaN, returns NULL instead. See policy notes for more on Nulling expressions.
  * `Log2Quiet(Expression* argument)` - the base 2 logarithm of the `argument`. See policy notes for more on Quiet expressions.
  * `LogNulling(Expression* base, Expression* argument)` - the base `base` logarithm of the `argument`. If the result would be infinity or NaN, returns NULL instead. See policy notes for more on Nulling expressions.
  * `LogQuiet(Expression* base, Expression* argument)` - the base `base` logarithm of the `argument`. See policy notes for more on Quiet expressions.
  * `Sin(Expression* argument)` - the sine of the `argument` (which is assumed to be in radians).
  * `Cos(Expression* argument)` - the cosine of the `argument` (which is assumed to be in radians).
  * `Tan(Expression* argument)` - the tangent of the `argument` (which is assumed to be in radians).
  * `Cot(Expression* argument)` - the cotangent of the `argument` (which is assumed to be in radians).
  * `Asin(Expression* argument)` - the arcsine (in radians) of the `argument`.
  * `Acos(Expression* argument)` - the arccosine (in radians) of the `argument`.
  * `Atan(Expression* argument)` - the arctangent (in radians) of the `argument`.
  * `Atan2(Expression* x, Expression* y)` - the arctangent in two variables - the arctangent of `y/x`, with the signs of `x` and `y` used to determine the quadrant of the result.
  * `Sinh(Expression* argument)` - the hyperbolic sine of the `argument`.
  * `Cosh(Expression* argument)` - the hyperbolic cosine of the `argument`.
  * `Tanh(Expression* argument)` - the hyperbolic tangent of the `argument`.
  * `Asinh(Expression* argument)` - the hyperbolic arcsine (the inverse hyperbolic sine) of the `argument`.
  * `Acosh(Expression* argument)` - the hyperbolic arccosine (the inverse hyperbolic cosine) of the `argument`.
  * `Atanh(Expression* argument)` - the hyperbolic arctangent (the inverse hyperbolic tangent) of the `argument`.
  * `ToDegrees(Expression* radians)` - `radians` to degrees conversion.
  * `ToRadians(Expression* degrees)` - `degrees` to radians conversion.
  * `Abs(Expression* argument)` - the absolute value of the `argument`. The output type is the same as the input type, except in the case of signed integers, for which the result is an unsigned integer (to handle the MININT case correctly).
  * `Round(Expression* argument)` - round to the nearest integer. The output type is the same as the input type.
  * `RoundToInt(Expression* argument)` - round to the next not larger integer. The output type is the same as the input type for integers, it is INT64 for floating point inputs. Note that garbage is produced if the `argument` is out of INT64 range.
  * `Floor(Expression* argument)` - round to the largest integer not larger than `argument`. The output type is the same as the input type.
  * `FloorToInt(Expression* argument)` - round to the largest integer not larger than argument. The output type is the same as the input type for integers, it is INT64 for floating point inputs. Note that garbage is produced if the `argument` is out of INT64 range.
  * `Ceil(Expression* argument)` - round to the smallest integer not smaller than `argument`. The output type is the same as the input type.
  * `CeilToInt(Expression* argument)` - round to the smallest integer not smaller than `argument`. The output type is the same as the input type for integers, it is INT64 for floating point inputs. Note that garbage is produced if the `argument` is out of INT64 range.
  * `Trunc(Expression* argument)` - round towards zero. The output type is the same as the input type.
  * `SqrtSignaling(Expression* argument)` - return the square root of `argument`. Fails if the `argument` is negative. See policy notes for more on Signaling expressions.
  * `SqrtNulling(Expression* argument)` - return the square root of `argument`. Returns NULL if the `argument` is negative. See policy notes for more on Nulling expressions.
  * `SqrtQuiet(Expression* argument)` - return the square root of `argument`. Returns NaN if the `argument` is negative. See policy notes for more on Quiet expressions.
  * `PowerSignaling(Expression* base, Expression* exponent)` - return base to the power `exponent`. Fails if the `base` is non-positive and the `exponent` is not a positive integer. See policy notes for more on Signaling expressions.
  * `PowerNulling(Expression* base, Expression* exponent)` - return `base` to the power `exponent`. Returns NULL if the `base` is non-positive and the `exponent` is not a positive integer. See policy notes for more on Nulling expressions.
  * `PowerQuiet(Expression* base, Expression* exponent)` - return `base` to the power `exponent`. Returns NaN or Inf if the `base` is non-positive and the `exponent` is not a positive integer. See policy notes for more on Quiet expressions.


## String expressions ##
String operation expressions. All of them return NULL if any of the inputs is NULL.

  * `ToString(Expression* argument)` - the STRING representation of `argument`. The DATEs are represented as YYYY/MM/DD, DATETIMEs as YYYY/MM/DD-HH:MM:SS, BOOLs as "TRUE" or "FALSE", DATA\_TYPEs as their textual representation, floating point numbers are given appropriate precision (via `DoubleToBuffer`), BINARYs are printed in hex.
  * `Concat(ExpressionList* arguments)` - returns a string concatenating the inputs. If some of the inputs are not of STRING type, they are first converted (as in `ToString`). The `arguments` list can have any length.
  * `Length(Expression* str)` - returns the length of `str`.
  * `Ltrim(Expression* str)` - returns a string that is `str` with all the leading whitespace removed.
  * `Rtrim(Expression* str)` - returns a string that is `str` with all the trailing whitespace removed.
  * `Trim(Expression* str)` - returns a string that is `str` with all the leading and trailing whitespace removed.
  * `ToUpper(Expression* str)` - returns `str` with all characters converted to uppercase.
  * `ToLower(Expression* str)` - returns `str` with all characters converted to lowercase.
  * `TrailingSubstring(Expression* str, Expression* pos)` - returns a trailing substring of `str` starting at `pos` (1-indexed). Negative `pos` is interpreted as "count from the end" - so `TrailingSubstring("Cow", -1) = "w"`. `pos` is expected to be any integer.
  * `Substring(Expression* str, Expression* pos, Expression* length)` - returns a substring of `str` starting at `pos` (`1-indexed`) and at most length bytes long. Negative `pos` is interpreted as "count from the end" - so `Substring("Cow", -1, 2) = "w"`. `pos` and `length` are expected to be any integers.
  * `StringOffset(Expression* haystack, Expression* needle)`- return the first index (1-indexed) that is a beginning of `needle` in `haystack`, or zero if `needle` does not appear as a substring in `haystack`.
  * `StringContains(Expression* haystack, Expression* needle)` - returns true iff `needle` is a substring of `haystack`.
  * `StringContainsCI(Expression* haystack, Expression* needle)` - the case insensitive version of StringContains.
  * `RegexpPartialMatch(Expression* str, StringPiece pattern)` - performs partial regular expression matching, using RE2, using the specified string argument. Returns true if matched, false if not matched.
  * `RegexpFullMatch(Expression* str, StringPiece pattern)` - performs full regular expression matching, using RE2, using the specified string argument.  Returns true if matched, false if not matched.
  * `RegexpReplace(Expression* str, StringPiece needle, Expression* substitute)` - replace all occurrences of needle in haystack with substitute. Needle can be a regular expression.
  * `RegexpExtract(Expression* str, StringPiece pattern)` - return the first substring of str matching pattern. If pattern cannot be matched into str, returns NULL.