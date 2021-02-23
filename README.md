# NAME

Unit::Duration - Work-time unit duration conversion and canonicalization

# VERSION

version 1.0

[![test](https://github.com/gryphonshafer/Unit-Duration/workflows/test/badge.svg)](https://github.com/gryphonshafer/Unit-Duration/actions?query=workflow%3Atest)
[![codecov](https://codecov.io/gh/gryphonshafer/Unit-Duration/graph/badge.svg)](https://codecov.io/gh/gryphonshafer/Unit-Duration)

# SYNOPSIS

    use Unit::Duration;

    my $ud = Unit::Duration->new;

    my $x = $ud->canonicalize('4d 6h 4d 3h');
    # $x eq '8 days, 9 hrs'

    my $y = $ud->canonicalize('4d 6h 4d 3h', { compress => 1 } );
    # $y eq '1 week, 2 days, 1 hrs'

    my $z = $ud->canonicalize(
        '4d 6h 4d 3h',
        {
            intra_space => '',
            extra_space => ' ',
            pluralize   => 0,
            unit_type   => 'letter',
            compress    => 1,
        },
    );
    # $z eq '1w 2d 1h'

    my $hours = $ud->sum_as( hours => '2 days -6h' );
    # $hours == 42

    my $ud_fully_described_with_defaults = Unit::Duration->new(
        name  => 'default',
        table => q{
            y | yr  | year    =  4 qtrs
            q | qtr | quarter =  3 mons
            o | mon | month   =  4 wks
            w | wk  | week    =  5 days
            d | day           =  8 hrs
            h | hr  | hour    = 60 mins
            m | min | minute  = 60 secs
            s | sec | second
        },
        intra_space => ' ',
        extra_space => ', ',
        pluralize   => 1,
        unit_type   => 'short',
        compress    => 0,
    );

    my $canonical_table_string    = $ud->get_table_string('default');
    my $canonical_table_structure = $ud->get_table_structure('default');

    $ud->set_table( default => $canonical_table_string );
    $ud->set_table(
        partial_default => [
            {
                letter   => 'd',
                short    => 'day',
                long     => 'day',
                duration => '8 hrs',
            },
            {
                letter   => 'h',
                short    => 'hr',
                long     => 'hour',
            },
        ],
    );

    $ud->verify_table($canonical_table_string)    or die;
    $ud->verify_table($canonical_table_structure) or die;

    my $duration_string = $ud->canonicalize(
        '3d 6h 1d 2h',
        {
            intra_space => ' ',
            extra_space => ', ',
            pluralize   => 1,
            unit_type   => 'short',
            compress    => 0,
        },
        'default', # table name or $table string or table structure
    );

# DESCRIPTION

This class provides the ability to "canonicalize" time durations based on custom
time units and their relationships to each other.

As an illustrative example, let's say you're a project manager dealing with
work-time duration estimates for a set of tasks. These might be weeks, days,
hours, or some combination of these units and/or other units. In the context of
work-time duration, 1 day does not equal 24 hours. The standard convention is  1
day equals 8 hours and 1 week equals 5 days. However, this is not universal. In
France, for example, the work week is typically 35 hours, not 40.

Assuming the default/typical case, though, let's say you have a task that's
estimated to take 12 hours. You can represent that duration as "12 hours" or
"12 hrs" or "1 day, 4 hours" or "1.5 days" or "1d 4h" or any number of other
ways.

# TABLES

Exactly how many hours constitute a day and that "hrs" is the canonical
shortened form of "hours" is all setup with a duration table. A duration table
consists of rows of units and columns of data types. The following is the
default table (in string form):

    y | yr  | year    =  4 qtrs
    q | qtr | quarter =  3 mons
    o | mon | month   =  4 wks
    w | wk  | week    =  5 days
    d | day           =  8 hrs
    h | hr  | hour    = 60 mins
    m | min | minute  = 60 secs
    s | sec | second

Each unit must have a "letter" and "short" form and may have an optional "long"
form. Any unit without a "long" form will use its "short" form as such. (See
"day" for an example.) Each unit must conclude with a duration, which should
define the unit's duration relative to some lower unit. Ultimately, there needs
to be 1 and only 1 unit that is the "base" unit. In the default table, this is
"second".

For tables in string form, the separation of columns can be done using any non-
digit, non-letter character other than commas and semicolons. All spacing is
ignored. Tables in data form are an arrayref of hashrefs where each hashref
contains `letter`, `short`, optionally a `long`, and a `duration`.

    {
        letter   => 'd',
        short    => 'day',
        long     => 'day',
        duration => '8 hrs',
    }

Tables are parsed and stored by name. The default table is stored as "default".

# METHODS

## new

This method instantiates a Unit::Duration object. It requires no inputs, but it
can be supplied with a table (using the `name` and `table` keys) and any
number of settings. See ["SETTINGS"](#settings) below.

    my $ud = Unit::Duration->new;

    my $ud_fully_described_with_defaults = Unit::Duration->new(
        name  => 'default',
        table => q{
            y | yr  | year    =  4 qtrs
            q | qtr | quarter =  3 mons
            o | mon | month   =  4 wks
            w | wk  | week    =  5 days
            d | day           =  8 hrs
            h | hr  | hour    = 60 mins
            m | min | minute  = 60 secs
            s | sec | second
        },
        intra_space => ' ',
        extra_space => ', ',
        pluralize   => 1,
        unit_type   => 'short',
        compress    => 0,
    );

## canonicalize

This method requires a duration string to parse. It will return a canonicalized
string based on the settings and table used.

    my $x = $ud->canonicalize('4d 6h 4d 3h');
    # $x eq '8 days, 9 hrs'

It can optionally can accept settings overrides in a hashref. See
["SETTINGS"](#settings) below.

    my $y = $ud->canonicalize('4d 6h 4d 3h', { compress => 1 } );
    # $y eq '1 week, 2 days, 1 hrs'

    my $z = $ud->canonicalize(
        '4d 6h 4d 3h',
        {
            compress    => 1,
            unit_type   => 'letter',
            intra_space => '',
            extra_space => ' ',
        },
    );
    # $z eq '1w 2d 1h'

It can also optionally be provided a table by name or as a string or data
structure.

    my $duration_string = $ud->canonicalize(
        '3d 6h 1d 2h',
        {
            intra_space => ' ',
            extra_space => ', ',
            pluralize   => 1,
            unit_type   => 'short',
            compress    => 0,
        },
        'default', # table name or $table string or table structure
    );

## sum\_as

Thie method accepts a unit label and a duration string. It will return a number
representing the value of the duration as the unit.

    my $hours = $ud->sum_as( hours => '2 days -6h' );
    # $hours == 42

It can also optionally be provided a table by name or as a string or data
structure.

    my $hours = $ud->sum_as( hours => '2 days -6h', 'default' );

## set\_table

This method sets a table for later use. It requires a name string, which will
be used to label the table, and the table data as either a string or a data
structure.

    $ud->set_table( default => $canonical_table_string );
    $ud->set_table(
        partial_default => [
            {
                letter   => 'd',
                short    => 'day',
                long     => 'day',
                duration => [ 8, 'hrs' ],
            },
            {
                letter   => 'h',
                short    => 'hr',
                long     => 'hour',
            },
        ],
    );

## get\_table\_string

This method returns a "canonical" table string for a given table label.

    my $canonical_table_string = $ud->get_table_string('default');

The "canonical" table string is not necessarily exactly the same as the input
string used to create the table. It's a uniform string, but it can be fed back
into `set_table` and other methods if desired.

## get\_table\_structure

This method returns a table as a data structure: an arrayref of hashrefs.

    my $canonical_table_structure = $ud->get_table_structure('default');

## verify\_table

This method will accept a table either as a string or data structure and will
return a true or false (1 or 0) based on whether the table is considered valid.

# SETTINGS

Settings affect the way `canonicalize` formats its output.

## intra\_space

This is a string and represents the space between a unit's numeric value and its
text label.

## extra\_space

This is a string and represents the space between different units.

## pluralize

This is a boolean that sets whether units should be "pluralized" when they don't
have the value of 1. For example, if you input "2h", that will become "2 hrs"
if `pluralize` is true or "2 hr" if `pluralize` is false.

## unit\_type

This is the unit type to use for the unit label. This will be either "letter",
"short", or "long".

## compress

By default, if you provide `canonicalize` a string with repeated same units,
it will merge these values, but it will not shift values between units. For
example:

    my $x = $ud->canonicalize('4d 6h 4d 3h');
    # $x eq '8 days, 9 hrs'

By setting `compress` to a true value, `canonicalize` will shift values
between units. For example:

    my $y = $ud->canonicalize('4d 6h 4d 3h', { compress => 1 } );
    # $y eq '1 week, 2 days, 1 hrs'

# SEE ALSO

You can look for additional information at:

- [GitHub](https://github.com/gryphonshafer/Unit-Duration)
- [MetaCPAN](https://metacpan.org/pod/Unit::Duration)
- [GitHub Actions](https://github.com/gryphonshafer/Unit-Duration/actions)
- [Codecov](https://codecov.io/gh/gryphonshafer/Unit-Duration)
- [CPANTS](http://cpants.cpanauthors.org/dist/Unit-Duration)
- [CPAN Testers](http://www.cpantesters.org/distro/U/Unit-Duration.html)

# AUTHOR

Gryphon Shafer <gryphon@cpan.org>

# COPYRIGHT AND LICENSE

This software is Copyright (c) 2021 by Gryphon Shafer.

This is free software, licensed under:

    The Artistic License 2.0 (GPL Compatible)
