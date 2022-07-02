# Mutexes are full of shit

The analogy goes something like this. A mutex is like a toilet. Every
lock on a mutex is a door leading into the toilet.

## Prepare

Consider a program with a bunch of threads. They all run around doing
their shit, touching things, modifying state, its nasty. Sooner or
later one thread needs to use the toilet, maybe it wants to write the
number two. It can enter the toilet (from any door) only if the toilet
is not currently in use. As it enters, it immediately counts as the
toilet being in use.

## Organize

Other threads also want to use the toilet. They start knocking on the
doors. Depending on the locking mechanism of the doors, the threads
might stand there knocking for quite a while before either falling
asleep or walk away. (I guess those threads did not need to use the
toilet as much as they initially thought they did huh?) Threads that
fall asleep either get woken up by a searing signal as the toilet
becomes vacant, or use an alarm clock set such that the thread gets
woken and attempt going through the door again after which the process
might repeat itself.

Some toilets allow the using thread to lock the door multiple times,
even other doors! This all happens from the inside, of course, since
as soon as the thread opens a door other threads are allowed to
enter. Making sure that the thread remembers to unlock all the doors
in the correct reverse order sometimes leads to stalling, trapping the
thread forever in its own filth and potentially ruining the pants of
other threads. As not all toilets are built with privacy and
soundproofing in mind, it could also be the case that it simply is
possible to leave the toilet without using the door at all.

## Observe

A toilet that allows multiple threads to enter simultaneously usually
does this by forcing all entering threads to promise not to write
anything to the toilet, no matter what the number is. They can however
all stand there and observe its content. Maybe some prior thread
forgot to flush?

What is common in all of these scenarios are that the thread using the
toilet does not even actually have to be on the toilet. It can sit in
there solving Rubik's cube or anything else completely unrelated to
the actual toilet. There are neither any limits to what a thread can
do in there nor how much time they spend. No matter what the thread
does, the size and complexity of the door is the same.

## Ponder

Toilets are important but doors are a problem. Some suggestions for
avoiding using doors are:

- Have the doors removed. If the threads can schedule their usage of
  the toilet some other way then why even have doors?

- Have the threads wear diapers. Let them clean up after themselves.

- Have personalized toilets. If every thread has their own toilet
  there is no need for any locking to be done.

- Have a go outside in nature. In the wild, although doors are rare,
  threads somehow manage to write numbers anyway. It is my belief that
  writing numbers outside is not only more satisfying, it is also
  faster. Think about it, who wants to be seen with their pointer
  visible?

I end this shitty story with a question to the reader: how many doors
do you want to your toilet?
