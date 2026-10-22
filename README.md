# NAME

Perl::Critic::Policy::ProhibitMisplacedMethods - A method goes in the class whose subclasses all need it, no higher and no lower.

# VERSION

version 0.001

# Perl::Critic::Policy::ProhibitMisplacedMethods

A class hierarchy works when each method is in the class whose subclasses all
need it.  A method that only one subclass needs, in the class above it, is
inherited by every other subclass for nothing.  Two subclasses that each define
the same method, under a class that defines none, are two copies of one thing,
and the next subclass makes a third.

Where a method is used is not a question that one file can answer.  So this
policy reads the whole distribution around the file that it critiques, through
[Perl::Critic::Distribution](https://metacpan.org/pod/Perl%3A%3ACritic%3A%3ADistribution).  Another policy that reads the distribution
through the same library shares the parse.

It reports two things, each at the definition of the sub that is misplaced.

- A sub that only one subclass uses

    A sub of a class, when everything in `bin/` and `lib/` that uses it is one
    subclass, or that subclass and the classes below it.  Move it down into that
    subclass.

- A sub that two subclasses define, and the class above them does not

    A sub of a class, when another class of the distribution defines a sub of the
    same name, and the nearest class that both inherit from defines none, and
    neither does any class above that one.  Move it up into that class.

## PROHIBITED

```perl
package My::Base;
sub helper { ... }                       # only My::Base::One uses it

package My::Base::One;
use parent -norequire, 'My::Base';
sub run { my ($self) = @_; return $self->helper }

package My::Base::Two;
use parent -norequire, 'My::Base';
sub release_command { ... }              # My::Base::Three has one too,
                                         # and My::Base has none

package My::Base::Three;
use parent -norequire, 'My::Base';
sub release_command { ... }
```

## ALLOWED

```perl
package My::Base;
sub hook { return () }                   # a default that a subclass can override
sub run  { my ($self) = @_; return $self->hook }

package My::Base::One;
use parent -norequire, 'My::Base';
sub hook { return 'one' }
```

A sub that the class itself uses, or that anything outside its subclasses uses,
is an interface.  It stays where it is, however few subclasses override it.

## WHAT COUNTS AS A USE

Every use is matched by its name alone, because the class behind `$obj` is not
known statically.  So this policy counts more uses than there are, and that
makes it report less, never more.

- A method call, `$obj->helper`, `$self->SUPER::helper` or
`Some::Class->helper`, and a method call inside an interpolating string,
as `"@{[ $obj->helper ]}"`.
- A function call, bare or qualified: `helper()`, `My::Base::helper()`.
- A reference: `\&helper`.
- A string or a `qw{}` word that spells the name, as in
`$self->can('helper')` or a dispatch table of names.
- A definition of the same name in a subclass, which overrides it.

A use counts only from a file in `bin/` or `lib/`.  A call from a test does
not make a subclass need a sub.  A use inside the sub itself, which is
recursion, does not count.

A sub that nothing uses is not reported.  That is
[Perl::Critic::Policy::ProhibitUnusedDefinitions](https://metacpan.org/pod/Perl%3A%3ACritic%3A%3APolicy%3A%3AProhibitUnusedDefinitions).

## PARENTS

A class names its parents in one of these ways:

```perl
use parent qw{My::Base};
use parent -norequire, 'My::Base';
use base 'My::Base';
our @ISA = ('My::Base');
push @ISA, 'My::Base';
extends 'My::Base';                      # Moose, Moo and Mouse
```

## CAVEATS

A class with more than one parent, or below one, is never reported, because
the class to move a sub to is not a single class.

A sub is reported as defined twice only when every class above both
definitions is in the distribution.  A class above them in another
distribution can define the sub, and this policy reads only this one.

A subclass in another distribution can use a sub of this one, and so can a
template.  Neither is read.  So a sub that a plugin outside the distribution
uses, and that one subclass in it uses, is reported.  That is what
`allow_subs` and `## no critic` are for.

A method name built at runtime, as in `$obj->$name`, is not read.  The
string that holds the name usually is, as a string.

The distribution is read once per process.  A file edited after that is not
seen again in that process.  ["THE CACHE ON DISK" in Perl::Critic::Distribution](https://metacpan.org/pod/Perl%3A%3ACritic%3A%3ADistribution#THE-CACHE-ON-DISK)
is how the next process sees it.

## CONFIGURATION

- `allow_subs`

    Space separated subs that are never reported, as a bare name or qualified with
    its package.  Adds to the built-in list, which is the names that perl or a
    framework calls for you, such as `DESTROY`, `import` and `BUILD`:

    ```perl
    [ProhibitMisplacedMethods]
    allow_subs = new My::Base::release_command
    ```

- `cache`

    Whether to keep what was read on disk between runs.  On by default.

- `cache_dir`

    Where it is kept.  The default is that of [Perl::Critic::Distribution](https://metacpan.org/pod/Perl%3A%3ACritic%3A%3ADistribution).  Give
    every policy that reads the distribution the same one, or none, so that they
    share a parse.

## METHODS

### supported\_parameters

`allow_subs`, the names that are never reported, added to the built-in list.
`cache` and `cache_dir`, as [Perl::Critic::Distribution](https://metacpan.org/pod/Perl%3A%3ACritic%3A%3ADistribution) keeps them.

### initialize\_if\_enabled

Folds the built-in exemptions back into whatever was configured, and registers
`collect` with [Perl::Critic::Distribution](https://metacpan.org/pod/Perl%3A%3ACritic%3A%3ADistribution), so that it runs in the same
parse as what any other enabled policy needs.

### default\_severity

SEVERITY\_LOW

### default\_themes

maintenance

### applies\_to

PPI::Statement::Sub

### violates

Standard [Perl::Critic::Policy](https://metacpan.org/pod/Perl%3A%3ACritic%3A%3APolicy) interface.  Returns a violation for a sub
that only one subclass uses, and one for a sub that another class defines
where the class above both does not.

### subs\_in

```perl
my $subs = $policy->subs_in($doc);
```

The distribution of a [Perl::Critic::Document](https://metacpan.org/pod/Perl%3A%3ACritic%3A%3ADocument), and the package and the name
of each sub that it defines, keyed by the address of the statement:
`{ dist => $dist, by_elem => { $address => [ $package, $name ] } }`.
Undef for a file outside `bin/` and `lib/` of a distribution.

It keeps the answer for the document that it was last asked about, because
`violates` asks once for each sub of a file.

## FUNCTIONS

These are how the policy reads a distribution.  A caller has no need of them,
and they are documented for whoever changes the policy.

### index\_of

```perl
my $index = index_of($dist);
```

What `collect` found in each file of `bin/` and `lib/` of a
[Perl::Critic::Distribution](https://metacpan.org/pod/Perl%3A%3ACritic%3A%3ADistribution), put together:
`{ parents => { $package => [ @parents ] }, defines => { $package => { $sub => 1 } }, users => { $sub => { $package => { $in_sub => 1 } } } }`.
`$in_sub` is the qualified name of the sub that the use is in, or the empty
string.

### ancestry

```perl
my ( $chain, $outside ) = ancestry( $index, $package );
```

`$package` and each class above it, nearest first, as far as the distribution
declares them.  `$outside` is true when the last of them has a parent that
the distribution does not declare.  An empty list when a class on the way has
more than one parent, or when the parents loop.

### used\_only\_below

```perl
my $subclass = used_only_below( $index, $package, $sub );
```

The subclass of `$package` that every use of its sub is in, or in a class
below that subclass.  A definition of the same name in a subclass counts as a
use.  Undef when `$package` itself or anything outside its subclasses uses
the sub, and when nothing uses it.

### defined\_beside

```perl
my ( $above, @others ) = defined_beside( $index, $package, $sub );
```

The nearest class above `$package` that another definition of the sub shares
with it, and the classes that have that definition.  An empty list when there
is none, when a class above `$package` defines the sub, and when a class above
it is outside the distribution.

### collect

```perl
my $found = collect($ppi);
```

What one [PPI::Document](https://metacpan.org/pod/PPI%3A%3ADocument) declares and uses, as plain data that JSON can hold:
`{ packages => { $package => { parents => [ ... ], subs => [ ... ] } }, uses => [ [ $package, $in_sub, $name ], ... ] }`.
[Perl::Critic::Distribution](https://metacpan.org/pod/Perl%3A%3ACritic%3A%3ADistribution) calls it for each file.

### walk

```perl
my $found = walk($ppi);
```

What `collect` returns, before it is plain data, and `defs`: each sub
statement, with its package and its name.  A lexical sub is not a method, and
is left out.  ["walk" in Perl::Critic::Distribution](https://metacpan.org/pod/Perl%3A%3ACritic%3A%3ADistribution#walk) says which package and which
sub each element is in.

### parents\_named

```perl
my @parents = parents_named($statement);
```

The classes that a statement names as the parents of its package, if it is
one that does: `use parent`, `use base`, an assignment or a `push` to
`@ISA`, or `extends`.

### names\_used

```perl
my @names = names_used($token);
```

Each name that a token can use: the name that a method or function call calls,
the last part of it when it is qualified, the name of a `\&name`, a string or
`qw{}` word that spells a name, and each method that an interpolating string
calls.

# BUGS

Please report any bugs or feature requests on the bugtracker website
[https://github.com/Troglodyne-Internet-Widgets/perl-critic-policy-prohibitmisplacedmethods/issues](https://github.com/Troglodyne-Internet-Widgets/perl-critic-policy-prohibitmisplacedmethods/issues)

When submitting a bug or request, please include a test-file or a
patch to an existing test-file that illustrates the bug or desired
feature.

# AUTHORS

Current Maintainers:

- George S. Baugh <george@troglodyne.net>

# COPYRIGHT AND LICENSE

Copyright (c) 2026 Troglodyne LLC

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:
The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.
THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
