# Module 6 --- Virtual Memory

## College-Level Notes \| Operating Systems

> **Exam priority: Very High**\
> The most important numerical topics are TLB/EMAT, page-fault overhead,
> FIFO/LRU/Optimal page replacement, and Belady's anomaly.

## Learning Objectives

By the end of this module, you should be able to:

-   Explain virtual memory, demand paging, and locality of reference.
-   Calculate Effective Memory Access Time (EMAT) with a TLB.
-   Explain page faults and calculate effective access time with
    page-fault probability.
-   Apply FIFO, LRU, and Optimal page-replacement algorithms.
-   Identify Belady's anomaly and explain stack algorithms.
-   Explain thrashing, working sets, and frame allocation.
-   Describe practical replacement approaches such as NRU, Second
    Chance, Clock, and LFU.
-   Compare paging, segmentation, and segmented paging.

------------------------------------------------------------------------

## 6.1 Why Virtual Memory?

**Virtual memory** allows a process to execute without keeping its
entire address space in physical RAM at the same time. The operating
system and hardware map virtual addresses to physical memory and use
secondary storage when needed.

### Why it is useful

-   A process can have a virtual address space larger than the currently
    available physical memory.
-   Only needed pages must be loaded, reducing unnecessary I/O.
-   More processes may be kept active at once.
-   Each process can use its own virtual address space, improving
    isolation and protection.

Virtual memory does not make storage as fast as RAM. A page fault that
requires disk or SSD I/O can be thousands to millions of times slower
than a normal memory access.

### Demand paging

**Demand paging** loads a page into RAM when it is first needed, rather
than loading every page before execution.

Typical flow:

1.  The CPU generates a virtual address.
2.  The MMU/TLB and page table are used to translate it.
3.  If the page is resident and access is permitted, execution
    continues.
4.  If the page is valid but not resident, a page-fault exception
    transfers control to the OS.
5.  The OS locates the page, obtains a free frame or selects a victim,
    and loads the page.
6.  The OS updates the page table and restarts the faulting instruction.

### Locality of reference

Demand paging works well because programs often access a relatively
small set of pages during a short period.

-   **Temporal locality:** A recently accessed item is likely to be
    accessed again soon. Example: a loop counter used repeatedly.
-   **Spatial locality:** Nearby addresses are likely to be accessed
    soon. Example: traversing an array sequentially.
-   **Sequential locality** is a common spatial-locality pattern in
    which memory is accessed in order.

### Translation and page-fault overview

``` text
CPU generates virtual address
          |
          v
      TLB lookup
       /      \
     Hit      Miss
      |         |
      |      Page-table lookup
      |         |
      +----+----+
           |
    Is page resident
    and access allowed?
       /          \
     Yes           No
      |             |
 Translate       Page-fault trap
      |             |
      v          OS handles fault
 Physical access     |
                 Load/update/restart
```

A TLB miss is not automatically a page fault. The page may be present in
RAM but its translation may simply not be cached in the TLB.

------------------------------------------------------------------------

## 6.2 Translation Lookaside Buffer (TLB)

A **TLB** is a small, fast cache that stores recent
virtual-page-to-physical-frame translations.

### Why it is needed

Without a TLB, a simple page-table translation may require: 1. A memory
access to read the page-table entry. 2. Another memory access to read or
write the requested data.

The TLB allows most references to avoid the page-table memory access
when the translation is cached.

### TLB hit and miss

-   **TLB hit:** Translation is found in the TLB.
-   **TLB miss:** Translation is not found in the TLB, so the system
    performs a page-table lookup (in a simple model).
-   **Page fault:** The referenced page is not resident, or another
    fault condition occurs. A TLB miss and a page fault are different
    events.

### Standard EMAT formula

Assume: - (t) = TLB lookup time - (m) = one main-memory access time -
(h) = TLB hit ratio - No page faults - Serial TLB lookup and a
single-level page table in memory

On a hit:

\[ T\_{`\text{hit}`{=tex}}=t+m \]

On a miss:

\[ T\_{`\text{miss}`{=tex}}=t+2m \]

Therefore:

\[
```{=tex}
\begin{aligned}
EMAT&=h(t+m)+(1-h)(t+2m)\\
&=\boxed{t+(2-h)m}
\end{aligned}
```
\]

### Solved numerical 1

Given: - TLB lookup time (t=20) ns - Memory access time (m=100) ns - Hit
ratio (h=0.80)

Hit time:

\[ 20+100=120`\text{ ns}`{=tex} \]

Miss time:

\[ 20+100+100=220`\text{ ns}`{=tex} \]

Effective memory access time:

\[
```{=tex}
\begin{aligned}
EMAT&=0.8(120)+0.2(220)\\
&=96+44\\
&=\boxed{140\text{ ns}}
\end{aligned}
```
\]

**Important:** TLB lookup time is included in both hit and miss cases in
this serial-lookup model.

### Solved numerical 2: find hit ratio

A system has (t=10) ns, (m=80) ns, and measured EMAT (=106) ns. Find
(h).

\[ 106=10+(2-h)80 \]

\[ 96=160-80h \]

\[ 80h=64 \]

\[ `\boxed{h=0.80=80\%}`{=tex} \]

### Alternative hardware timing models

Some problems assume the TLB lookup overlaps with memory access rather
than occurring serially. In such a model, the formula changes. **Always
use the timing assumptions stated in the question.** The formula above
is for serial lookup.

------------------------------------------------------------------------

## 6.3 Page-Fault Handling

A **page fault** occurs when an address translation or memory access
cannot be completed normally. A common case is a valid virtual page that
is currently not resident in physical memory.

### Page-fault handling steps

1.  The CPU references a page that is not resident.
2.  Hardware raises a page-fault exception/trap.
3.  The OS checks whether the address and requested access are legal.
4.  If illegal, the OS reports an error or terminates the process.
5.  If legal, the OS locates the page on storage or determines how to
    create it.
6.  The OS obtains a free frame; if none is available, it selects a
    victim page.
7.  If the victim is dirty, it may need to be written back.
8.  The required page is read or created in the selected frame.
9.  The page table and relevant TLB state are updated.
10. The faulting instruction is restarted.

A page fault does not always require disk I/O. Some faults can be
resolved using zero-filled pages, copy-on-write, or pages already cached
in memory.

### Page-fault effective access time

Let: - (p) = probability that a memory reference causes a page fault -
(m) = normal memory access time - (F) = additional page-fault service
time, excluding the normal access counted separately

Then:

\[ `\boxed{EMAT=m+pF}`{=tex} \]

This is a simplified model. If a problem defines its fault-service time
differently, avoid double-counting or omitting the normal memory access.

### Solved numerical 3

Given: - (m=100) ns - (p=0.001) - (F=8) ms (=8,000,000) ns

\[
```{=tex}
\begin{aligned}
EMAT&=100+(0.001)(8,000,000)\\
&=100+8,000\\
&=\boxed{8,100\text{ ns}=8.1\ \mu s}
\end{aligned}
```
\]

The page-fault overhead dominates normal access time even though only 1
in 1,000 references faults.

### Solved numerical 4: maximum acceptable page-fault rate

Normal access time is 100 ns and additional page-fault service time is 5
ms. Find the maximum (p) if EMAT must not exceed 150 ns.

\[ 150=100+p(5,000,000) \]

\[ p=`\frac{50}{5,000,000}`{=tex} =`\boxed{10^{-5}}`{=tex} \]

The page-fault probability must be no greater than (10\^{-5}), under
this model.

### Unit conversions

-   (1 `\mu `{=tex}s=1,000) ns
-   (1 ms=1,000 `\mu `{=tex}s=1,000,000) ns
-   Convert all terms to the same unit before calculating EMAT.

------------------------------------------------------------------------

## 6.4 Combined TLB and Page-Fault Numericals

First calculate normal EMAT from the TLB model, then add expected
page-fault overhead if the question defines fault service time as
additional.

\[ `\boxed{EMAT_{\text{overall}}\approx t+(2-h)m+pF}`{=tex} \]

Assumptions: serial TLB lookup, one-level page table, (F) is additional
fault cost, and page-fault probability (p) is defined per memory
reference. More detailed models may separate TLB misses, faults,
dirty-page write-back, and restart costs.

### Solved numerical 5

Given: - (t=10) ns - (m=100) ns - (h=0.95) - (p=10\^{-5}) - (F=4) ms
(=4,000,000) ns

Normal EMAT:

\[ EMAT_0=10+(2-0.95)(100)=115`\text{ ns}`{=tex} \]

Page-fault overhead:

\[ pF=10\^{-5}`\times4`{=tex},000,000=40`\text{ ns}`{=tex} \]

Approximate overall EMAT:

\[ `\boxed{115+40=155\text{ ns}}`{=tex} \]

### Common mistake

Do not add the page-fault service time for every reference. Multiply it
by the page-fault probability. Also ensure that the fault probability
and fault cost use compatible assumptions.

------------------------------------------------------------------------

## 6.5 Page Replacement Algorithms

When a page fault occurs and no free frame is available, the OS must
choose a resident page to remove. The chosen page is called the **victim
page**.

-   If a victim page is dirty, writing it back can add overhead.
-   If it is clean and backed by a file or storage copy, it may be
    discarded without writing it back.
-   The goal of a replacement algorithm is usually to reduce page faults
    or total cost.

### Reference string

A **reference string** is the sequence of page numbers requested by a
process.

Example:

``` text
7, 0, 1, 2, 0, 3, 0, 4
```

The first reference to a page that is not in memory causes a fault. If
frames start empty, the first reference is always a page fault.

### A. FIFO --- First In, First Out

FIFO removes the page that has been resident for the longest time.

-   Maintain pages in arrival order.
-   On a fault with full frames, evict the oldest resident page.
-   Simple to implement.
-   May evict a frequently used page just because it is old.
-   Can exhibit Belady's anomaly.

### B. Optimal (OPT / MIN)

OPT removes the page whose next use is farthest in the future.

-   Produces the minimum possible number of page faults for a known
    reference string and fixed number of frames.
-   Requires knowledge of future references, so it is mainly a
    theoretical benchmark.
-   Does not exhibit Belady's anomaly.

### C. LRU --- Least Recently Used

LRU removes the page that has not been used for the longest time in the
past.

-   Uses recent history as a prediction of future use.
-   Can be implemented using timestamps, a stack/list, or
    approximations.
-   Does not exhibit Belady's anomaly.
-   Exact LRU can require significant tracking overhead.

### Replacement comparison

  Algorithm   Victim choice                 Uses               Belady's anomaly?
  ----------- ----------------------------- ------------------ -------------------
  FIFO        Oldest loaded page            Arrival order      Possible
  OPT         Farthest next use in future   Future knowledge   No
  LRU         Least recently used page      Past history       No

------------------------------------------------------------------------

## 6.6 Worked Page-Replacement Numerical

Use this reference string:

\[ 7, 0, 1, 2, 0, 3, 0, 4, 2, 3, 0, 3, 2, 1, 2, 0, 1, 7, 0, 1 \]

Number of frames = 3, initially empty.

### FIFO trace

`F` means page fault; `H` means hit.

    Reference   Frame 1   Frame 2   Frame 3 Result
  ----------- --------- --------- --------- --------
            7         7       ---       --- F
            0         7         0       --- F
            1         7         0         1 F
            2         2         0         1 F
            0         2         0         1 H
            3         2         3         1 F
            0         2         3         0 F
            4         4         3         0 F
            2         4         2         0 F
            3         4         2         3 F
            0         0         2         3 F
            3         0         2         3 H
            2         0         2         3 H
            1         1         2         3 F
            2         1         2         3 H
            0         1         0         3 F
            1         1         0         3 H
            7         1         0         7 F
            0         1         0         7 H
            1         1         0         7 H

Total FIFO page faults:

\[ `\boxed{15}`{=tex} \]

### LRU and OPT

For the same reference string and 3 frames, the correct counts are:

-   FIFO: **15 faults**
-   LRU: **12 faults**
-   OPT: **9 faults**

When solving manually: 1. Draw one row per reference. 2. Mark hits
without replacing a page. 3. On each fault, apply only the chosen
algorithm's rule. 4. Count all faults, including the first reference.

**Important:** Do not mix FIFO arrival order with LRU recency order.
They select victims differently.

------------------------------------------------------------------------

## 6.7 Belady's Anomaly and Stack Algorithms

### Belady's anomaly

Belady's anomaly occurs when increasing the number of page frames causes
**more** page faults for an algorithm and reference string.

It can occur with FIFO.

### Classic example

Reference string:

\[ 1, 2, 3, 4, 1, 2, 5, 1, 2, 3, 4, 5 \]

FIFO produces: - 3 frames: 9 page faults - 4 frames: 10 page faults

Therefore, adding one frame increases the number of faults from 9 to 10:
Belady's anomaly.

### Stack algorithms

An algorithm has the **stack property** if the set of pages resident
with (n) frames is always a subset of the set resident with (n+1)
frames, for the same reference history.

-   LRU has the stack property.
-   OPT has the stack property.
-   FIFO does not generally have it.

Therefore, LRU and OPT do not exhibit Belady's anomaly.

### GATE trap

Do not assume that more frames always reduce page faults for every
algorithm. The guarantee depends on the algorithm. FIFO can show
Belady's anomaly; LRU and OPT cannot.

------------------------------------------------------------------------

## 6.8 Frame Allocation

When several processes run concurrently, the OS must decide how many
physical frames each process receives.

### Equal allocation

If (m) frames are shared among (n) processes, equal allocation gives
approximately:

\[ `\boxed{\frac{m}{n}\text{ frames per process}}`{=tex} \]

### Proportional allocation

A common proportional-allocation model gives a process with size (s_i):

\[ `\boxed{a_i=\frac{s_i}{\sum_j s_j}\times m}`{=tex} \]

where: - (a_i) = frames allocated to process (i) - (s_i) = size or page
demand of process (i) - (m) = total frames available

**Example:** There are 120 frames and three processes of sizes 10,000,
20,000 and 30,000 pages.

Total size:

\[ 10,000+20,000+30,000=60,000 \]

Allocations:

\[ a_1=`\frac{10,000}{60,000}`{=tex}`\times120`{=tex}=20 \]

\[ a_2=`\frac{20,000}{60,000}`{=tex}`\times120`{=tex}=40 \]

\[ a_3=`\frac{30,000}{60,000}`{=tex}`\times120`{=tex}=60 \]

Check:

\[ 20+40+60=`\boxed{120\text{ frames}}`{=tex} \]

### Local vs global replacement

-   **Local replacement:** A process replaces one of its own resident
    pages. Its replacement decisions do not directly take frames from
    other processes.
-   **Global replacement:** A process can choose a victim page belonging
    to another process from the global pool. This can change how many
    frames other processes have available.

Global replacement can improve overall utilization but may let one
process's memory pressure affect another process.

------------------------------------------------------------------------

## 6.9 Thrashing

**Thrashing** occurs when the system spends so much time handling page
faults and moving pages that little useful execution is completed.

### Why it happens

1.  Too many processes compete for physical frames.
2.  Processes receive fewer frames than their active memory references
    require.
3.  Pages are repeatedly evicted and needed again soon.
4.  Page faults and storage I/O increase sharply.
5.  CPU utilization can fall because processes wait for paging I/O.
6.  The OS may mistakenly increase multiprogramming in response to low
    CPU utilization, making the problem worse.

### Degree of multiprogramming

``` text
More processes initially
        |
        v
Higher CPU utilization
        |
        v
Memory becomes overcommitted
        |
        v
Too few frames per process
        |
        v
Frequent page faults / paging I/O
        |
        v
THRASHING
        |
        v
CPU utilization can drop
```

### Working-set model

The **working set** of a process is the set of distinct pages referenced
during a recent window of execution.

Let:

\[
W(t,`\Delta`{=tex})=`\text{set of pages referenced during the most recent window }`{=tex}`\Delta`{=tex}
\]

Working-set size:

\[
`\boxed{|W(t,\Delta)|=\text{number of distinct pages in the window}}`{=tex}
\]

**Example:** The recent page-reference window is:

\[ 2, 3, 2, 5, 6, 3, 5, 8, 6, 2 \]

Distinct pages are:

\[ {2,3,5,6,8} \]

So the working-set size is:

\[ `\boxed{5\text{ pages}}`{=tex} \]

If the process is allocated substantially fewer than five frames in this
example, it may repeatedly fault on pages in its working set. Actual
thrashing depends on workload and other system conditions, not only this
one count.

### Working-set frame demand

If several processes have working-set sizes
(W_1,W_2,`\ldots`{=tex},W_n), the total demand is:

\[ `\boxed{D=\sum_{i=1}^{n}|W_i|}`{=tex} \]

If (D) exceeds the number of frames available for these processes, the
system cannot hold all their current working sets simultaneously. The OS
may reduce the degree of multiprogramming by suspending one or more
processes.

**Example:** Working sets are 180, 220 and 150 frames.

\[ D=180+220+150=550`\text{ frames}`{=tex} \]

If only 500 frames are available, the shortfall is:

\[ 550-500=`\boxed{50\text{ frames}}`{=tex} \]

### Page-Fault Frequency (PFF)

PFF is another approach to controlling frame allocation.

-   If a process's fault frequency is too high, allocate more frames if
    possible.
-   If its fault frequency is sufficiently low, the OS may reclaim some
    frames.
-   If memory is overcommitted, suspending a process may be necessary.

\[
`\boxed{\text{Page-fault rate}=\frac{\text{Number of page faults}}{\text{Number of references}}}`{=tex}
\]

Percentage:

\[
`\boxed{\text{PFF percentage}=\frac{\text{Faults}}{\text{References}}\times100\%}`{=tex}
\]

**Example:** 40 faults occur in 2,000 references.

\[ PFF=`\frac{40}{2000}`{=tex}=0.02=`\boxed{2\%}`{=tex} \]

Whether 2% is too high depends on the OS policy, storage latency, and
workload.

------------------------------------------------------------------------

## 6.10 Practical Page-Replacement Implementations

### NRU --- Not Recently Used

NRU classifies pages using two bits: - (R): referenced recently - (M):
modified/dirty

    Class   R   M Meaning
  ------- --- --- --------------------------------
        0   0   0 Not recently referenced, clean
        1   0   1 Not recently referenced, dirty
        2   1   0 Recently referenced, clean
        3   1   1 Recently referenced, dirty

NRU selects a victim from the lowest-numbered non-empty class. Within a
class, selection may be arbitrary.

### Second Chance

Second Chance is FIFO with a reference-bit check.

1.  Inspect the oldest candidate.
2.  If its R bit is 0, replace it.
3.  If its R bit is 1, clear the bit and give the page another chance;
    continue to the next candidate.

### Clock algorithm

Clock implements Second Chance with frames arranged in a circular list
and a moving hand.

1.  If the current page has R=0, replace it.
2.  If R=1, clear R and advance the hand.
3.  Repeat until a page with R=0 is found.

### LRU implementation ideas

-   **Timestamp/counter:** Record the last access time and replace the
    page with the oldest timestamp.
-   **Stack/list:** Move a page to the most-recently-used position on
    every reference; evict from the least-recently-used end.
-   Exact tracking can be expensive, so practical systems often use
    approximations.

### LFU --- Least Frequently Used

LFU replaces the page with the smallest reference count.

Example: - P1 count = 8 - P2 count = 3 - P3 count = 6 - P4 count = 3

P2 and P4 are tied for the lowest count. A tie-breaking rule, such as
LRU, is needed to select between them.

**Caution:** A simple LFU counter can preserve pages that were popular
long ago but are no longer useful. Implementations may use aging or
decay.

------------------------------------------------------------------------

## 6.11 Segmentation and Segmented Paging

### Segmentation

Segmentation divides a process into logically meaningful, variable-sized
regions such as code, data, and stack.

A logical address is:

\[ (`\text{segment number}`{=tex},`\text{offset}`{=tex}) \]

Each segment-table entry stores a base and limit. The offset must be
within the segment's limit.

\[ `\boxed{PA=\text{base}+\text{offset}}`{=tex} \]

provided:

\[ `\boxed{0\leq\text{offset}<\text{limit}}`{=tex} \]

**Example:** Segment base = 4,000; limit = 800 bytes; offset = 300.

Since (300\<800):

\[ PA=4000+300=`\boxed{4300}`{=tex} \]

Offset 800 is invalid when the limit is 800 and offsets start at zero.

### Segmented paging

Segmented paging divides each logical segment into fixed-size pages.
Translation uses: 1. Segment number to find the segment's page table. 2.
Page number to find the physical frame. 3. Offset to identify a byte
inside the frame.

For logical address ((s,p,d)), if page (p) of segment (s) maps to frame
(f):

\[ `\boxed{PA=(f\times\text{page size})+d}`{=tex} \]

### Paging vs segmentation

  -----------------------------------------------------------------------
  Feature                 Paging                  Segmentation
  ----------------------- ----------------------- -----------------------
  Division                Fixed-size pages        Variable-size logical
                                                  segments

  Logical address         Page number + offset    Segment number + offset

  Mapping                 Page table              Segment table with base
                                                  and limit

  External fragmentation  Avoided for fixed-size  Can occur with
                          frames                  variable-sized segments

  Internal fragmentation  Possible in final page  Not inherent to
                                                  variable-size segments
                                                  in the same way

  Programmer visibility   Usually transparent     Often reflects logical
                                                  program structure
  -----------------------------------------------------------------------

Segmented paging combines logical segmentation with page-based physical
allocation. It can avoid external fragmentation in the physical
allocation of pages, while maintaining separate logical segments.

------------------------------------------------------------------------

## 6.12 Formula Sheet

  Quantity                                    Formula
  ------------------------------------------- -----------------------------------------------------------------------
  TLB hit time                                (t+m)
  TLB miss time (simple single-level model)   (t+2m)
  EMAT with TLB                               (h(t+m)+(1-h)(t+2m)=t+(2-h)m)
  EMAT with page faults                       (m+pF), if (F) is additional fault time
  Combined TLB + faults                       (t+(2-h)m+pF), under stated assumptions
  Page-fault rate                             Faults ÷ references
  PFF percentage                              ((`\text{faults}`{=tex}/`\text{references}`{=tex})`\times100`{=tex}%)
  Working-set size                            Number of distinct pages in recent window
  Total working-set demand                    Sum of working-set sizes
  Proportional frame allocation               ((s_i/`\sum `{=tex}s_j)`\times `{=tex}m)
  Segmentation physical address               Base + offset, if offset \< limit
  Segmented paging physical address           Frame × page size + offset

------------------------------------------------------------------------

## 6.13 Common GATE and College-Exam Traps

1.  **TLB miss is not a page fault.** A page can be resident even when
    its translation is absent from the TLB.
2.  **Include TLB lookup time in both cases** for the serial-lookup
    timing model.
3.  **Check the problem's EMAT model.** Some questions assume overlap,
    while others assume serial access.
4.  **Convert units before calculating.** Convert milliseconds or
    microseconds to nanoseconds if normal memory access is in ns.
5.  **Multiply page-fault service time by fault probability.** Do not
    add a full fault-service cost to every reference.
6.  **Count cold-start faults.** With initially empty frames, the first
    reference to each distinct page faults until it is loaded.
7.  **Do not mix replacement rules.** FIFO uses arrival order; LRU uses
    past recency; OPT uses future references.
8.  **Belady's anomaly can occur with FIFO.** LRU and OPT are stack
    algorithms and do not exhibit it.
9.  **More frames do not guarantee fewer faults for every algorithm.**
    FIFO is the standard counterexample.
10. **Thrashing is about excessive paging relative to useful
    execution.** High page-fault activity alone is not sufficient to
    diagnose it without context.
11. **Working set counts distinct pages, not references.** Repeated
    references to the same page count once in the set.
12. **For NRU, prefer the lowest non-empty class.** The lowest class is
    not necessarily the page with the smallest page number.
13. **In segmentation, test the limit first.** An offset equal to the
    limit is out of bounds.
14. **In segmented paging, use the physical frame number**, not the
    segment or page number, in the physical-address formula.

------------------------------------------------------------------------

## 6.14 Practice Questions

Try these without looking at the worked examples.

### Conceptual questions

1.  Explain virtual memory and demand paging.
2.  Differentiate temporal and spatial locality.
3.  Distinguish a TLB miss from a page fault.
4.  Explain the page-fault handling routine.
5.  Compare FIFO, LRU and Optimal page replacement.
6.  Define Belady's anomaly and explain why FIFO can exhibit it.
7.  Explain the stack property of LRU and OPT.
8.  Explain thrashing and the working-set model.
9.  Compare local and global page replacement.
10. Explain NRU, Second Chance, Clock, and LFU.
11. Differentiate paging, segmentation and segmented paging.
12. Explain why a high page-fault rate can sharply increase effective
    access time.

### Numerical questions

**Q1. TLB EMAT:** TLB lookup time = 10 ns, memory access = 80 ns, hit
ratio = 90%. Calculate EMAT under serial lookup.

**Q2. Find hit ratio:** TLB lookup time = 5 ns, memory access = 100 ns,
measured EMAT = 125 ns. Find the TLB hit ratio.

**Q3. Page faults:** Memory access time = 100 ns, fault probability =
(10\^{-5}), additional fault service time = 5 ms. Calculate EMAT.

**Q4. Combined EMAT:** TLB time = 10 ns, memory time = 100 ns, hit ratio
= 95%, fault probability = (10\^{-5}), additional fault service time = 4
ms. Calculate approximate overall EMAT.

**Q5. FIFO/LRU/OPT:** With 3 frames and initially empty memory,
calculate page faults for each algorithm for the reference string:

`1, 2, 3, 1, 4, 2, 5, 1, 2, 3, 4, 5`

**Q6. Belady's anomaly:** For FIFO and the reference string
`1, 2, 3, 4, 1, 2, 5, 1, 2, 3, 4, 5`, calculate page faults with 3 and 4
frames.

**Q7. Working set:** The recent reference window is
`1, 2, 1, 3, 4, 2, 5, 3, 1`. Find the distinct pages and working-set
size.

**Q8. Working-set demand:** Three processes need working sets of 200,
250 and 180 frames. The system has 600 frames available for these
processes. Calculate total demand and the surplus or shortfall.

**Q9. PFF:** A process experiences 60 page faults in 3,000 references.
Calculate its page-fault rate and percentage.

**Q10. Frame allocation:** Allocate 120 frames proportionally to
processes with page demands of 10,000, 20,000 and 30,000.

**Q11. NRU:** Pages A, B, C, and D have (R,M) bits of (1,1), (0,1),
(0,0), and (1,0), respectively. Classify each and identify the preferred
replacement class.

**Q12. Segmentation:** Segment base = 7,000, limit = 1,000, offset =
750. Find the physical address and state whether offset 1,000 is valid.

------------------------------------------------------------------------

## Final Revision Checklist

-   [ ] Explain demand paging and locality.
-   [ ] Calculate TLB hit, miss, and EMAT.
-   [ ] Calculate page-fault overhead using consistent units.
-   [ ] Trace FIFO, LRU and OPT accurately.
-   [ ] Identify Belady's anomaly and stack algorithms.
-   [ ] Calculate working-set size and total frame demand.
-   [ ] Explain thrashing and PFF-based responses.
-   [ ] Classify pages for NRU and trace Second Chance/Clock.
-   [ ] Solve frame-allocation numericals.
-   [ ] Translate segmentation and segmented-paging addresses.

**Highest-priority revision:** TLB/EMAT, page-fault probability,
FIFO/LRU/OPT traces, Belady's anomaly, and working-set calculations.
