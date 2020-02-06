# NAME

Debug::Filter::PrintExpr - Convert comment lines to debug print statements

# SYNOPSIS

```perl
    use Debug::Filter::PrintExpr;

    my $s = 'a scalar';
    my @a = qw(this is an array);
    my %h = (key1 => 'value1', key2 => 'value2', '' => 'empty', undef => undef);
    my $ref = \%h;
    

    #${$s}
    #@{@a}
    #%{ %h }
    #${ calc: @a * 2 }
    #\{$ref}
```

This program produces an output like this:

```perl
    line 13: $s = 'a scalar';
    line 14: @a = ('this', 'is', 'an', 'array');
    line 15: %h = ('' => 'empty', 'key1' => 'value1', 'key2' => 'value2', 'undef' => undef);
    calc: @a * 2  = 8;
    line 17: dump($ref);
    $_[0] = {
              '' => 'empty',
              'key1' => 'value1',
              'key2' => 'value2',
              'undef' => undef
            };
```

# DESCRIPTION

## The Problem

Providing debug output often results in a couple of print statements that
display the value of some expression and some kind of description.
When the program development is finished, these statements must be
made conditional on some variable or turned into comments.

Often the contents of arrays or hashes need to be presented in a
readable way, leading to repeated lines of similar code.

C programmers use the preprocessor to solve this problem.
As Perl has it's own filter mechanism for preprocessing,
this leads to a similar solution in Perl.

## A Solution

The [Filter::Simple](https://metacpan.org/pod/Filter::Simple) module by Damian Conway provides a convenient way
of implementing Perl filters.

`Debug::Filter::PrintExpr` makes use of [Filter::Simple](https://metacpan.org/pod/Filter::Simple)
to transform specially formed comment lines into print statements
for various debugging purposes.
(Besides, there is [Smart::Comments](https://metacpan.org/pod/Smart::Comments) from Damian, that does something
very similar but more advanced.)

Just by removing the "use" of `Debug::Filter::PrintExpr` completely,
disabling it partially by

```
    no Debug::Filter::PrintExpr;
```

or making the usage conditional (e.g. on an environment variable)
by

```perl
    use if $ENV{DEBUG}, 'Debug::Filter::PrintExpr';
```

all these lines (or a part of them) lose their magic and remain
simple comments.

The comment lines to be transformed must follow this format:

\# _sigil_ { \[_label_:\] \[_expression_\] }

or more formally must be matched by the following regexp:

```
qr{
       ^\h*\#
       (?<type>[%@\$\\"#])
       \{\h*
       (?<label>[[:alpha:]_]\w*:)?
       \h*
       (?<expr>\V+)?
       \}\h*$
}x
```

where `type` represents the sigil, `label` an optional label and
`expr` an optional expression.

If the label is omitted, it defaults to `line nnn:`, where nnn is the
line number in the program.

The sigil determines the evaluation context for the given expression
and the output format of the result:

- `$`

    The expression is evaluated in scalar context. Strings are printed
    inside single quotes, integer and floating point numbers are
    printed unquoted and dual valued variables are shown in both
    representations seperated by a colon.
    Undefined values are represented by the unquoted string `undef`.
    Hash and array references are shown in their usual string representation
    as e.g. `ARRAY(0x19830d0)` or `HASH(0xccba88)`.
    Blessed references are shown by the class they are belong to as
    `blessed(_class_)`.

- `@`

    The expression is evaluated in list context and the elements of the
    list are printed like single scalars, separated by commas and gathered
    in parentheses.

- `%`

    The expression is used as argument in a while-each loop and the output
    consists of pairs of the form 'key' => _value_ inside parentheses.
    _value_ is formatted like a single scalar.

- `\`

    The expression shall evaluate to a list of references.
    These will be evaluated using [Data::Dumper](https://metacpan.org/pod/Data::Dumper) as if used as
    parameter list to a subroutine call, i.e. named as `$_[_n_]`.

- `"`

    The expression is evaluated in scalar context as a string.

- `#`

    The expression is evaluated in scalar context as a numeric value.

The usage and difference between `#${}`, `#"{}` and `##{}` is
best described by example:

```perl
    my $dt = DateTime->now;
    #${$dt}         # line nn: $dt = blessed(DateTime);
    #"{$dt}         # line nn: $dt = '2019-10-27T15:54:28';

    my $num = ' 42 ';
    #${$num}        # line nn: $num = ' 42 ';
    $num + 0;
    #${$num}        # line nn: $num = ' 42 ' : 42;
    #"{$num}        # line nn: $num = ' 42 ';
    ##{$num}        # line nn: $num = 42;
```

The forms #${}, #"{}, ##{} and #@{} may be used for any type of expression
and inside the #%{} form, arrays are permitted too.
With the varibles $s, @a and %h as defined above, it is possible
to use:

```
    #@{scalar_as_array: $s}
    #${array_as_scalar :@a}
    #@{hash_as_array: %h}
    #%{array_as_hash: @a}
```

and produce these results:

```perl
    scalar_as_array: $s = ('this is a scalar');
    array_as_scalar: @a = 4;
    hash_as_array: %h = ('k1', 'v1', 'k2', 'v2');
    array_as_hash: @a = ('0' => 'this', '1' => 'is', '2' => 'an', '3' => 'array');
    
```

Regular expressions may be evaluated too:

```
    #@{"a<b>c<d><e>f<g>h" =~ /\w*<(\w+)>/g}
```

gives:

```
    line nn: "a<b>c<d><e>f<g>h" =~ /\w*<(\w+)>/g = ('b', 'd', 'e', 'g');
```

If the expression is omitted, only the label will be printed.
The sigil `$` should be used in this case.

Requirements for the expression are:

- It must be a valid Perl expression.
- In case of the #%{}-form, it must be a valid argument to the
each() builtin function, i.e. it should resolve to an array or hash.

A PrintExpr will be resolved to a block and therefore may be located
anywhere in the program where a block is valid. 
Do not put it in a place, where a block is required (e.g. after a
conditional) as this would break the code when running without the
filter.

As a code snippet of the form `{label: expr}` is a valid perl
expression and the generated code will result in a 
braced expression, a simple consistency check can be done by removing
hash and sigil from the PrintExpr line:
The resulting code must still be valid and should only emit a warning
about a useless use of something in void context.

## Usage

The `use` statement for `Debug::Filter::PrintExpr` may contain
arguments as described in [Exporter::Tiny::Manual::Importing](https://metacpan.org/pod/Exporter::Tiny::Manual::Importing).
Importable functions are `isnumeric` and `isstring` as well
as the import tag `:all` for both of them.

The (optional) global options hash may contain
these module specific entries:

- debug => 1

    This option causes the resulting source code after comment
    transformation to be written to `STDERR`.
    This option may also be specified as `-debug` in the
    `use` statement.

- nofilter => 1

    This options disables source code filtering if only the import
    of functions is desired.
    This option may also be specified as `-nofilter` in the
    `use` statement.

## Functions

- `isstring(_$var_)`

    This function returns true if the "string slot" of _$var_ has a value.
    This is the case when a string value was assigned to the variable,
    the variable has been used (recently) in a string context
    or when the variable is dual-valued.

    It will return false for undefined variables, references and
    variables with a numeric value that have never been used in a
    string context.

- `isnumeric(_$var_)`

    This function returns true if the "numeric slot" if _$var_ has a
    value.
    This is the case when a numeric value (integer or floating point) was
    assigned to the variable, the variable has been used (recently) in a
    numeric context or when the variable is dual-valued.

    It will return false for undefined variables, references and variables
    with a string value that have never been used in numeric context.

## Variables

- `$Debug::Filter::PrintExpr::handle`

    The filehandle that is referenced by this variable is used for
    printing the generated output.
    The default is STDERR and may be changed by the caller.

# SEE ALSO

Damian Conway's module [Smart::Comments](https://metacpan.org/pod/Smart::Comments) provides something similar
and more advanced.

While [Smart::Comments](https://metacpan.org/pod/Smart::Comments) has lots of features for visualizing the
program flow, this module focuses on data representation.
The main requirements for this module were:

- Always print the source line number or a user provide label.
- Always print the literal expression along with its evaluation.
- Give a defined context where the expression is evaluated.
Especially provide scalar and list context or perform an iteration
over a while-each-loop.
The usage of [Data::Dumper](https://metacpan.org/pod/Data::Dumper) was adopted later from Damian's
implementation.
- Trailing whitespace in values should be clearly visible.
- Distinguish between the numeric and string value of a variable.
- undefined values should be clearly distinguishable from empty values.

The first three requirements are not met by [Smart::Comments](https://metacpan.org/pod/Smart::Comments) as there is
an extra effort needed to display a line number,
the display of a label and the literal expression are mutual exclusive
and a specific context is not enforced by the module.

All in all, the module presented here is not much more than a
programming exercise.

Importing the functions `isstring` and `isnumeric` is done
by [Exporter::Tiny](https://metacpan.org/pod/Exporter::Tiny).
For extended options see [Exporter::Tiny::Manual::Importing](https://metacpan.org/pod/Exporter::Tiny::Manual::Importing).

Other related modules: [Scalar::Util](https://metacpan.org/pod/Scalar::Util), [Data::Dumper](https://metacpan.org/pod/Data::Dumper)

# AUTHOR

Jörg Sommrey

# LICENCE AND COPYRIGHT

Copyright (c) 2018-2020, Jörg Sommrey. All rights reserved.

This program is free software; you can redistribute it and/or modify it
under the terms of either: the GNU General Public License as published
by the Free Software Foundation; or the Artistic License.

See [http://dev.perl.org/licenses/](http://dev.perl.org/licenses/) for more information.
