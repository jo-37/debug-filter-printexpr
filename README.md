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

This produces an output like:

```perl
    L13: $s = 'a scalar';
    L14: @a = ('this', 'is', 'an', 'array');
    L15: %h = ('' => 'empty', 'key1' => 'value1', 'key2' => 'value2', 'undef' => undef);
    calc: @a * 2 = 8;
    L17: dump($ref);
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

or making the usage conditional (e.g. on environment variable DEBUG)
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

If the label is omitted, it defaults to `L_n_:`, where n is the
line number in the program.

The sigil determines the evaluation context for the given expression
and the output format of the result:

- `$`

    The expression is evaluated in scalar context. Strings are printed
    inside single quotes, integer and floating point numbers are
    printed unquoted and dual valued variables are shown in the form
    `dualvar(_numval_, '_stringval_')`.
    Undefined values are represented by the unquoted string `undef`.
    Hash and array references are shown in their usual string representation
    as e.g. `ARRAY(0x19830d0)` or `HASH(0xccba88)`.
    Blessed references are shown by the class they are belong to as
    `blessed(class)`.

- `"`

    The expression is evaluated in scalar context as a string.

- `#`

    The expression is evaluated in scalar context as a numeric value.

- `@`

    The expression is evaluated in list context and the elements of the
    list are printed like single scalars, separated by commas and gathered
    in parentheses.

- `%`

    The expression is evaluated as a list of key-value pairs
    and is presented in the form 'key' => _value_,... inside parentheses.
    _value_ is formatted like a single scalar.

- `\`

    The expression shall evaluate to a list of references.
    These will be evaluated using [Data::Dumper](https://metacpan.org/pod/Data::Dumper) and named
    like parameters in a subroutine, i.e. `$_[_n_]`.

The usage and difference between `#${}`, `#"{}` and `##{}` is
best described by example:

```perl
    my $dt = DateTime->now;
    #${$dt}         # Ln: $dt = blessed(DateTime);
    #"{$dt}         # Ln: $dt = '2019-10-27T15:54:28';

    my $num = ' 42 ';
    #${$num}        # Ln: $num = ' 42 ';
    $num + 0;
    #${$num}        # Ln: $num = dualvar(42, ' 42 ');
    #"{$num}        # Ln: $num = ' 42 ';
    ##{$num}        # Ln: $num = 42;
```

The forms #${}, #"{}, ##{} and #@{} may be used for any type of expression
and inside the #%{} form, arrays are permitted too.
With the varibles $s, @a and %h as defined above, it is possible
to use:

```
    #@{scalar_as_array: $s}
    #${array_as_scalar :@a}
    #@{hash_as_array: %h}
```

and produce these results:

```
    scalar_as_array: $s = ('this is a scalar');
    array_as_scalar: @a = 4;
    hash_as_array: %h = ('k1', 'v1', 'k2', 'v2');
    
```

Regular expressions may be evaluated too:

```
    #@{"a<b>c<d><e>f<g>h" =~ /\w*<(\w+)>/g}
```

gives:

```
    Ln: "a<b>c<d><e>f<g>h" =~ /\w*<(\w+)>/g = ('b', 'd', 'e', 'g');
```

If the expression is omitted, only the label will be printed.
The sigil `$` should be used in this case.

Requirements for the expression are:

- It must be a valid Perl expression.
- In case of the #%{}-form, it must evaluate to a list of pairs, e.g.
a hash.

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
following arguments:

- -debug

    This option causes the resulting source code after comment
    transformation to be written to `STDERR`.

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
over the key-value pairs of a hash.
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

Other related modules: [Scalar::Util](https://metacpan.org/pod/Scalar::Util), [Data::Dumper](https://metacpan.org/pod/Data::Dumper)

# AUTHOR

Jörg Sommrey

# LICENCE AND COPYRIGHT

Copyright (c) 2018-2020, Jörg Sommrey. All rights reserved.

This program is free software; you can redistribute it and/or modify it
under the terms of either: the GNU General Public License as published
by the Free Software Foundation; or the Artistic License.

See [http://dev.perl.org/licenses/](http://dev.perl.org/licenses/) for more information.
