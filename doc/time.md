% Old-UTC, new-UTC, and TAI seconds and dates

Prior to 1972, UTC used variable-duration seconds, as determined by
the time it takes the Earth to rotate by a certain angle.

TAI used exact seconds as determined by how long it takes light to
travel a certain distance in a vacuum.

UTC seconds were generally longer than TAI seconds by a tiny margin,
since Earth slowed down and lagged behind.  So time-stamps in UTC
(like "1970-01-01 00:00:00") would generally be reached later, by a
couple seconds, than in TAI.

In 1972, there was approximately a 10-second difference between the
two systems, i.e. during approximately UTC 1972-01-01 00:00:00, the
corresponding TAI time-stamp for the moment was 1972-01-01 00:00:10.

They decided to declare the difference to be exactly ten seconds, and
started counting forth in constant-duration seconds in both systems,
using the constant-duration TAI seconds.

However this meant that wall clocks would get out of sync with Earth's
rotation, so that for example when your clock hits midnight, the Earth
hasn't finished a round yet.  UTC accommodated this via so-called leap
seconds: on certain dates, clocks would tick up to 23:59:60 before
reaching 00:00:00 of the next day, giving Earth a second to catch up.

TAI time-stamps don't have these 61-second minutes, so they again run
ahead of UTC time more and more, but at least by discreet seconds.
Until 2014, there have been 25 leap seconds, so together with the
10-second difference in 1972, TAI time-stamps are now 35 seconds ahead
of UTC time-stamps, as of the time I'm writing this.

That's not where it ends though.  UTC time-stamps prior to 1972 are
very complex, since the real-time duration between two time-stamps
which have a one-second difference is neither an exact second, nor
always the same physical duration.  For this reason, pre-1972 UTC
time-stamps were, in the computing world, *discarded*, and replaced
with time-stamps one would get by counting back exact TAI seconds from
the year 1972.

So the moment which we now label as UTC 1970-01-01 00:00:00 would
actually had been labeled 1970-01-01 00:00:02 in original UTC.

This is why, to avoid ambiguity, Unix time can be said to be *the
number of TAI seconds elapsed since UTC 1972-01-01 00:00:00, plus the
amount of TAI seconds in two TAI years, minus the amount of leap
seconds since 1972*.  But most just say it's the number of seconds
since UTC 1970-01-01 00:00:00, implicitly referring to the retrofitted
UTC time-stamp, implicitly using TAI seconds, and generally forgetting
to add the "minus leap seconds" part.

***

That's actually mostly fine, since the ambiguity of pre-1972 UTC
time-stamps is generally irrelevant.  But what really hurts is that
people assume Unix time to monotonically increase by 1 with each
ticking TAI second, when in fact during leap seconds it waits two TAI
seconds to increase by 1.

(That is a historical decision for making it easy to convert from Unix
time-stamps to UTC time-stamps without taking into account leap
seconds, at the expense of some time-stamps (integers) not being
unambiguously convertible to a UTC time; specifically seconds like
23:59:60 in UTC are not distinguishable from the preceding 23:59:59,
since both are represented by the same integer in Unix time.)

For this reason, R7RS Scheme adopted a time-stamp format that yields
the amount of TAI seconds elapsed since TAI 1970-01-01 00:00:00,
meaning it generally returns a higher integer than Unix time.

It was considered to return the number of TAI seconds since TAI
1970-01-01 00:00:10 instead, which corresponds to the retrofitted UTC
1970-01-01 00:00:00, but that was seen as unnecessary (even making
things more complicated in a sense) so it was dropped.

Fin.
