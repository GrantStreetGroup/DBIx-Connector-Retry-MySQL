# NAME

DBIx::Connector::Retry::MySQL - MySQL-specific DBIx::Connector with retry support

# VERSION

version v1.0.1

# SYNOPSIS

    my $conn = DBIx::Connector::Retry::MySQL->new(
        connect_info  => [ 'dbi:Driver:database=foobar', $user, $pass, \%args ],
        retry_debug   => 1,
        timer_options => {
            # Default options from Algorithm::Backoff::RetryTimeouts
            max_attempts          => 8,
            max_actual_duration   => 50,
            jitter_factor         => 0.1,
            timeout_jitter_factor => 0.1,
            adjust_timeout_factor => 0.5,
            min_adjust_timeout    => 5,
            # ...among others
        },
    );

    # Keep retrying/reconnecting on errors
    my ($count) = $conn->run(ping => sub {
        $_->do('UPDATE foobar SET updated = 1 WHERE active = ?', undef, 'on');
        $_->selectrow_array('SELECT COUNT(*) FROM foobar WHERE updated = 1');
    });

    my ($count) = $conn->txn(fixup => sub {
        $_->selectrow_array('SELECT COUNT(*) FROM barbaz');
    });

    # Plus everything else in DBIx::Connector::Retry and DBIx::Connector

# DESCRIPTION

DBIx::Connector::Retry::MySQL is a subclass of [DBIx::Connector::Retry](https://metacpan.org/pod/DBIx%3A%3AConnector%3A%3ARetry) that will
explicitly retry on MySQL-specific transient error messages, as identified by
[DBIx::ParseError::MySQL](https://metacpan.org/pod/DBIx%3A%3AParseError%3A%3AMySQL), using [Algorithm::Backoff::RetryTimeouts](https://metacpan.org/pod/Algorithm%3A%3ABackoff%3A%3ARetryTimeouts) as its retry
algorithm.  This connector should be much better at handling deadlocks, connection
errors, and Galera node flips to ensure the transaction always goes through.

It is essentially a DBIx::Connector version of [DBIx::Class::Storage::DBI::mysql::Retryable](https://metacpan.org/pod/DBIx%3A%3AClass%3A%3AStorage%3A%3ADBI%3A%3Amysql%3A%3ARetryable).

# INHERITED ATTRIBUTES

This inherits all of the attributes of [DBIx::Connector::Retry](https://metacpan.org/pod/DBIx%3A%3AConnector%3A%3ARetry):

## [connect\_info](https://metacpan.org/pod/DBIx%3A%3AConnector%3A%3ARetry#connect_info)

## [mode](https://metacpan.org/pod/DBIx%3A%3AConnector%3A%3ARetry#mode)

## [disconnect\_on\_destroy](https://metacpan.org/pod/DBIx%3A%3AConnector%3A%3ARetry#disconnect_on_destroy)

## max\_attempts

Unlike ["max\_attempts" in DBIx::Connector::Retry](https://metacpan.org/pod/DBIx%3A%3AConnector%3A%3ARetry#max_attempts), this is just an alias to the value in
["timer\_options"](#timer_options).

As such, it has a slightly adjusted default of 8.

## retry\_debug

Like [retry\_debug](https://metacpan.org/pod/DBIx%3A%3AConnector%3A%3ARetry#retry_debug), this turns on debug warnings for
retries.  But, this module has a bit more detail in the messages.

## retry\_handler

Since the whole point of the module is the retry-handling code, this attribute cannot be
set.

## failed\_attempt\_count

Unlike ["failed\_attempt\_count" in DBIx::Connector::Retry](https://metacpan.org/pod/DBIx%3A%3AConnector%3A%3ARetry#failed_attempt_count), this is just an alias to the
value in the internal timer object.

## [exception\_stack](https://metacpan.org/pod/DBIx%3A%3AConnector%3A%3ARetry#exception_stack)

# NEW ATTRIBUTES

## timer\_class

The class used for delay and timeout setting calculations.  By default, it's
[Algorithm::Backoff::RetryTimeouts](https://metacpan.org/pod/Algorithm%3A%3ABackoff%3A%3ARetryTimeouts), but you can use a sub-class of this, if you so
choose, provided that it has a similar interface.

## timer\_options

Controls all of the options passed to the timer constructor, using ["timer\_class"](#timer_class) as the
object.

## aggressive\_timeouts

Boolean that controls whether to use some of the more aggressive, query-unfriendly
timeouts:

- mysql\_read\_timeout

    Controls the timeout for all read operations.  Since SQL queries in the middle of
    sending its first set of row data are still considered to be in a read operation, those
    queries could time out during those circumstances.

    If you're confident that you don't have any SQL statements that would take longer than
    the timeout settings (or at least returning results before that time), you can turn this
    option on.  Otherwise, you may experience longer-running statements going into a retry
    death spiral until they hit the final timeout and die.

- wait\_timeout

    Controls how long the MySQL server waits for activity from the connection before timing
    out.  While most applications are going to be using the database connection pretty
    frequently, the MySQL default (8 hours) is much much longer than the mere seconds this
    engine would set it to.

Default is off.  Obviously, this setting makes no sense if `max_actual_duration`
within ["timeout\_options"](#timeout_options) is disabled.

## retries\_before\_error\_prefix

Controls the number of retries (not tries) needed before the exception message starts
using the statistics prefix, which looks something like this:

    Failed run coderef: Out of retries, attempts: 5 / 4, timer: 34.5 / 50.0 sec

The default is 1, which means a failed first attempt (like a non-transient failure) will
show a normal exception, and the second attempt will use the prefix.  You can set this to
0 to always show the prefix, or a large number like 99 to keep the exception clean.

## parse\_error\_class

The class used for MySQL error parsing.  By default, it's [DBIx::ParseError::MySQL](https://metacpan.org/pod/DBIx%3A%3AParseError%3A%3AMySQL), but
you can use a sub-class of this, if you so choose, provided that it has a similar
interface.

## enable\_retry\_handler

Boolean to enable the retry handler.  The default is, of course, on.  This can be turned
off to temporarily disable the retry handler.

# CAVEATS

## $dbh settings

See ["$dbh settings" in DBIx::Connector::Retry](https://metacpan.org/pod/DBIx%3A%3AConnector%3A%3ARetry#dbh-settings).

## Savepoints and nested transactions

See ["Savepoints and nested transactions" in DBIx::Connector::Retry](https://metacpan.org/pod/DBIx%3A%3AConnector%3A%3ARetry#Savepoints-and-nested-transactions).

## (Ab)using $dbh directly

See ["(Ab)using $dbh directly" in DBIx::Connector::Retry](https://metacpan.org/pod/DBIx%3A%3AConnector%3A%3ARetry#Ab-using-dbh-directly).

## Connection modes

Due to the caveats of ["Fixup mode" in DBIx::Connector::Retry](https://metacpan.org/pod/DBIx%3A%3AConnector%3A%3ARetry#Fixup-mode), `fixup` mode is changed to
just act like `no_ping` mode.  However, `no_ping` mode is safer to use in this module
because it comes with the same retry protections as the other modes.  Certain retries,
such as connection/server errors, come with an explicit disconnect to make sure it starts
back up with a clean slate.

In `ping` mode, the DB will be pinged on the first try.  If the retry explicitly
disconnected, the connector will simply connect back to the DB and run the code, without
a superfluous ping.

# AUTHOR

Grant Street Group <developers@grantstreet.com>

# COPYRIGHT AND LICENSE

This software is Copyright (c) 2020 - 2022 by Grant Street Group.

This is free software, licensed under:

    The Artistic License 2.0 (GPL Compatible)
