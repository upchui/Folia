<div align=center>
    <p>Fork of <a href="https://github.com/PaperMC/Paper">Paper</a> which adds regionised multithreading to the dedicated server.</p>
</div>

## Overview

Folia groups nearby loaded chunks to form an "independent region."
See [REGION_LOGIC.md](REGION_LOGIC.md) for exact details on how Folia
will group nearby chunks.
Each independent region has its own tick loop, which is ticked at the
regular Minecraft tickrate (20TPS). The tick loops are executed
on a thread pool in parallel. There is no main thread anymore, 
as each region effectively has its own "main thread" that executes
the entire tick loop.

For a server with many spread out players, Folia will create many
spread out regions and tick them all in parallel on a configurable sized
threadpool. Thus, Folia should scale well for servers like this.

Folia is also its own project, this will not be merged into Paper
for the foreseeable future. 

## Plugin compatibility

There is no more main thread. I expect _every_ single plugin
that exists to require _some_ level of modification to function
in Folia. Additionally, multithreading of _any kind_ introduces
possible race conditions in plugin held data - so, there are bound
to be changes that need to be made.

So, have your expectations for compatibility at 0.

## API plans

Currently, there is a lot of API that relies on the main thread. 
I expect basically zero plugins that are compatible with Paper to 
be compatible with Folia. However, there are plans to add API that 
would allow Folia plugins to be compatible with Paper.

For example, the Bukkit Scheduler. The Bukkit Scheduler inherently
relies on a single main thread. Folia's RegionisedScheduler and Folia's
EntityScheduler allow scheduling of tasks to the "next tick" of whatever
region "owns" either a location or an entity. These could be implemented
on regular Paper, except they schedule to the main thread - in both cases,
the execution of the task will occur on the thread that "owns" the
location or entity. This concept applies in general, as the current Paper
(single threaded) can be viewed as one giant "region" that encompasses
all chunks in all worlds. 

It is not yet decided whether to add this API to Paper itself directly
or to Paperlib.

### The new rules

The other important rule is that the regions tick in _parallel_, and not 
_concurrently_. They do not share data, they do not expect to share data,
and sharing of data _will_ cause data corruption. 
Code that is running in one region under no circumstance can 
be accessing or modifying data that is in another region. Just 
because multithreading is in the name, it doesn't mean that everything 
is now thread-safe. In fact, there are only a _few_ things that were 
made thread-safe to make this happen. As time goes on, the number 
of thread context checks will only grow, even _if_ it comes at a 
performance penalty - _nobody_ is going to use or develop for a 
server platform that is buggy as hell, and the only way to 
prevent and find these bugs is to make bad accesses fail _hard_ at the 
source of the bad access.

This means that Folia compatible plugins need to take advantage of 
API like the RegionisedScheduler and the EntityScheduler to ensure 
their code is running on the correct thread context.

In general, it is safe to assume that a region owns chunk data
in an approximate 8 chunks from the source of an event (i.e player
breaks block, can probably access 8 chunks around that block). But,
this is not guaranteed - plugins should take advantage of upcoming
thread-check API to ensure correct behavior.

The only guarantee of thread-safety comes from the fact that a
single region owns data in certain chunks - and if that region is
ticking, then it has full access to that data. This data is 
specifically entity/chunk/poi data, and is entirely unrelated
to **ANY** plugin data.

Normal multithreading rules apply to data that plugins store/access
their own data or another plugin's - events/commands/etc are called 
in _parallel_ because regions are ticking in _parallel_ (we CANNOT 
call them in a synchronous fashion, as this opens up deadlock issues 
and would handicap performance). There are no easy ways out of this, 
it depends solely on what data is being accessed. Sometimes a 
concurrent collection (like ConcurrentHashMap) is enough, and often a 
concurrent collection used carelessly will only _hide_ threading 
issues, which then become near impossible to debug.

### Current API additions

- RegionisedScheduler and EntityScheduler acting as a replacement for 
  the BukkitScheduler, however they are not yet fully featured.

### Current broken API

- Most API that interacts with portals / respawning players / some
  player login API is broken.
- ALL scoreboard API is considered broken (this is global state that
  I've not figured out how to properly implement yet)
- World loading/unloading
- Entity#teleport. This will NEVER UNDER ANY CIRCUMSTANCE come back, 
  use teleportAsync
- Could be more

### Planned API additions

- Proper asynchronous events. This would allow the result of an event
  to be completed later, on a different thread context. This is required
  to implement some things like spawn position select, as asynchronous
  chunk loads are required when accessing chunk data out-of-region.
- World loading/unloading
- TickThread#isTickThread overloads to API
- More to come here

### Planned API changes

- Super aggressive thread checks across the board. This is absolutely
  required to prevent plugin devs from shipping code that may randomly
  break random parts of the server in entirely _undiagnosable_ manners.
- More to come here