---
title: Baby, lemme alloc
publishDate: 2025-11-08 10:00:00
img: /assets/stock-1.jpg
img_alt: Iridescent ripples of a bright blue and pink liquid
description: |
  How I stopped fearing malloc and learned to love the heap.
tags:
  - Low-level
---
<a href="https://github.com/theotrama/my-malloc" target="_blank" rel="noopener noreferrer">
  View on GitHub
</a>


I recently set off on a journey to deepen my fundamental understanding of computers. As part of that process I decided to write code only in the C programming language for a while.
Like many developers, I used to be afraid of pointers. And of course of `malloc`. Once I understood how pointers actually work,
they started to make sense. But the heap and manual memory management? That still felt like black magic. Whenever I used `malloc`, `realloc` or `free` in my programs,
I sort of knew what to do, but I didn't fully grasp what was happening behind the scenes.
Thus, I started my journey into understanding the theory behind dynamic memory allocation and how glibc implements `malloc`.
That led me down the rabbit hole of writing my own basic memory allocator which I am pretty proud of.
There's already an amazing book that covers this topic, *Computer Systems: A Programmer's Perspective* (CS:APP).
Still, I think my explanation can help understand the theory behind simple memory management. And honestly once you have grasped the concept
you might not be able to write your own production grade `malloc` implementation but your understanding of what happens behind the scene will greatly improve.
You'll have a much better intuition for what's really happening when you call `malloc` or `free` in C.
In this short article I will cover two strategies for allocating and managing memory that build on each other.  I will start with an implicit free list (copying the implementation of CS:APP),
then bolt an explicit free list on it (for faster access). For the explicit free list I will also provide the code. In the next article, I will then focus on `malloc` and explain it conceptually.
For now, let's start with the simplest one.

## Implicit Free Lists
These are the most basic implementation of a memory allocator. Their name comes from the fact that from any block you can implicitly determine
where the next and previous blocks are located. Each block consists of a **header**, **payload**, and **footer**.
The header and footer store metadata about the block - specifically its **size** and whether it's **allocated**.

With that information you can traverse the memory like a linked list. From each block you can jump to its predecessor and successor by adding or subtracting the size of the next or previous block.
Let's make this more tangible and show it with an example. Below Figure 1 shows the simplest possible state of an implicit free list.
It features only the prologue and epilogue blocks and a 4 byte alignment padding at the start.
The prologue header's total size is 8 bytes, and it's allocated flag is set to 1. The epilogue header's size is set to 0 and its allocated flag is set to 1.

<figure id="figure-1">
  <img src="../../../public/assets/diagrams/explicit-free-list/implicit_free_list.svg" alt="Basic implicit free list">
  <figcaption><em>Figure 1: The most basic implicit free list</em></figcaption>
</figure>
<br><br>

### Extending the Heap, Allocating, and Freeing Memory
At this point our free list doesn't help us allocate memory. Right now the whole space of the list is taken up.
This is when we need to extend our heap. We initialized it with a size of 16 bytes. Now, we extend the heap by another 64 bytes to make space for user data.
Now can you guess how our list will look like? Exactly, now we have one more block but this time its allocated flag is set to 0 as it is still free (Figure 2).

<figure id="figure-2">
  <img src="../../../public/assets/diagrams/explicit-free-list/implicit_free_list-state-1.svg" alt="implicit free list">
  <figcaption><em>Figure 2: An implicit free list with a free block</em></figcaption>
</figure>
<br><br>


With that free block available, let us now finally allocate some memory and call `malloc(7)`. Can you guess how the list will look after this command? Consider that
each block must reserve space for the header and footer (which are each 4 bytes in size). The total required block size is thus
```
7 (payload) + 4 (header) + 4 (footer) = 15 bytes
```
As our blocks are aligned to be multiples of 8 bytes, we round up to **16 bytes**. Now figure 3 shows our free list:

<figure id="figure-3">
  <img src="../../../public/assets/diagrams/explicit-free-list/implicit_free_list-state-2.svg" alt="implicit free list">
  <figcaption><em>Figure 3: An implicit free list with one allocated and one free block</em></figcaption>
</figure>
<br><br>

### Implementing a Simple Allocator

With these concepts, you have everything needed to implement a simple allocator.
1. Store the pointer to the first block of the free list called the heap list pointer.
2. Traverse the list using the header and footer to find a suitable free block
3. When allocating (`malloc`), mark the chosen block as allocated by setting its flag to 1.

Common allocation strategies include:
* **First fit**: Use the first block that is large enough.
* **Next fit**: Similar to first fit, but instead of starting at the beginning of the list we continue where the last search has stopped.
* **Best fit**: Find a free block with size closest to that of the requested size.

### Freeing Memory and Coalescing
All right so now we know how to request a new block, but what if we need to `free` a block? Well simple: we pass the pointer we want to free to `free` and set the allocation flag of that block to 0.
However, there are some edge cases that we need to handle to avoid false fragmentation of the list. We need to ensure that every time we free a block we coalesce it with the previous and next blocks in case these are not allocated.
This is called  *immediate* coalescing. In case we clean the fragments up in the background it would be called *deferred* coalescing.
Okay so now we coalesce immediately. We have to deal with the following four cases:
1. Previous and next blocks are already allocated: easy, we just insert
2. Previous block free, next block allocated: Merge with the previous block
3. Previous block allocated, next block free: Merge with the next block
4. Previous and next block free: Merge with both neighbors

Figure 4 shows these cases in a real allocation scenario:
<figure id="figure-4">
  <img src="../../../public/assets/diagrams/explicit-free-list/implicit_free_list-edge-cases.svg" alt="implicit free list edge cases">
  <figcaption><em>Figure 4: Edge cases when freeing memory</em></figcaption>
</figure>
<br><br>

### Handling Large Allocations
With that we have covered all the possible freeing cases. But there is one more case we need to consider. What if we want to `malloc` a huge chunk of data that our current list cannot fit? Then, we simply request
more memory from the OS based on the size requested by `malloc`. Afterward, we need to coalesce as shown in figure 5:

<figure id="figure-5">
  <img src="../../../public/assets/diagrams/explicit-free-list/implicit_free_list-request-memory.svg" alt="Request more memory from the OS">
  <figcaption><em>Figure 5: Request more memory from the OS</em></figcaption>
</figure>
<br><br>

### Takeaways for implicit free lists

With that we have all the tools in our hand to build our own implicit list. I highly recommend implementing one yourself, it really helps to understand memory on a lower level. For guidance, check out chapter 9 of CS:APP.
I followed their implementation at first, copying it into my codebase to understand the details. Then I reimplemented it from scratch, referencing the original only when I got stuck.
Once comfortable, I added my own explicit free list on top, which made everything really click.

Next, we'll explore **explicit free lists**, which improve allocation speed and efficiency.


## Explicit free lists
As the name implies these lists are explicit. But why? Well, because while previously we were implicitly traversing our list based on the block sizes
now we store additional information in our list for the blocks that are free. Basically, we introduce a doubly linked list where each block explicitly knows what its
previous and next blocks are.

### Block layout changes
Recall that in an implicit free list each block has a header, a payload, and a footer. For an explicit free list we add two more fields inside the payload of free blocks: `prev` and `next`,
pointing to the previous and next free blocks, respectively.

The size to store these addresses depends on whether you're on a 32-bit or a 64-bit system. Nowadays, most modern machines
are 64-bit, so `prev`/`next` are typically 8 bytes each.
Figure 6 shows a free block in this layout: The prologue and epilogue blocks remain. The only thing that changed is that now each unallocated block holds information about
its previous and next free blocks. As there are currently no other free blocks available, both `prev` and `next` are `NULL`.
<figure id="figure-6">
  <img src="../../../public/assets/diagrams/explicit-free-list/explicit_free_list-block-definition.svg" alt="A block in an explicit free list">
  <figcaption><em>Figure 6: A block in an explicit free list</em></figcaption>
</figure>
<br><br>

### Example: Allocate three blocks, free two
Now, let's walk through a more complex scenario. In figure 7 you can see that we start again with the exact same list as before. We first `malloc` three times and then we `free` two of the previously
allocated blocks. When calling `free` we add them to our free list.
```c
void *p1 = malloc(8);
void *p2 = malloc(8);
void *p3 = malloc(8);

free(p1);
free(p3);
```
So now each time we need to insert a block we don't need to iterate over the entire list. We just take our explicit free list
and find the first fit there. This improves our allocation time.

<figure id="figure-7">
  <img src="../../../public/assets/diagrams/explicit-free-list/explicit_free_list-case-1.svg" alt="A more complex explicit free list">
  <figcaption><em>Figure 7: An explicit free list with two free entries</em></figcaption>
</figure>
<br><br>

### `malloc` with an explicit list
Basically, the code for the underlying implicit list stays the same, but now every time we `malloc` or `free` we have to ensure
that our explicit list stays up to date. For `malloc` this is pretty straight forward. All we have to do is to remove
the entry from our free list and ensure that the previous block now points to the next block. And the next block points to the previous block. Let's also illustrate
this with a simple graphic in figure 8. We start with the list from figure 7 and now we call `malloc(7)`. Then we typically:
1. Search the explicit free list for the first fit
2. Remove the chosen block from the free list
3. If there is a free part leftover, we insert it back into the explicit free list
4. Update the free list pointer if needed (if we removed the head of the explicit free list)

<figure id="figure-8">
  <img src="../../../public/assets/diagrams/explicit-free-list/explicit_free_list-malloc.svg" alt="Removing an element from the free list">
  <figcaption><em>Figure 8: Removing an element from the free list</em></figcaption>
</figure>
<br><br>

### `free` and coalescing with an explicit free list

The more complex cases are again the cases where we free memory. Here, we need to ensure that when we coalesce blocks in the underlying list
we need to also update our explicit free list. We have exactly the same cases as before. We only need to bolt some more logic onto them for the explicit free list management.
* Case 1 - previous and next blocks are already allocated: easy, we just insert and also insert the new block into the list
* Case 2 - previous block free, next block allocated: Coalesce the new and previous blocks. Nothing else to do. The previous block is already correctly linked in the list.
* Case 3 - previous block allocated, next block free: Coalesce the new and next blocks. Remove the next block from the free list. Insert the current block instead.
* Case 4 - previous and next block free: Coalesce the previous, new and next blocks. Remove the next block from the free list. Update the previous block with the information from the next block.

**Case 1**: previous and next blocks are already allocated. Just insert the new block into the free list and update the free list pointer.
<figure id="figure-9">
  <img src="../../../public/assets/diagrams/explicit-free-list/explicit_free_list-case-1.svg" alt="Case 1 - previous and next blocks are allocated">
  <figcaption><em>Figure 9: Case 1 - previous and next blocks are allocated</em></figcaption>
</figure>
<br><br>


**Case 2**: previous block is free and the next block is allocated. This is the simplest case. We only need to coalesce and that's it. No need to update the explicit free list
as the previous block is already correctly linked in the list.

<figure id="figure-10">
  <img src="../../../public/assets/diagrams/explicit-free-list/explicit_free_list-case-2.svg" alt="Case 2 - previous block free and next block allocated">
  <figcaption><em>Figure 10: Case 2 - previous block free and next block allocated</em></figcaption>
</figure>
<br><br>

**Case 3**: We coalesce the new block with the next block. We thus remove the next block from the free list and then insert the new block. The new block becomes
our new free list pointer.
<figure id="figure-11">
  <img src="../../../public/assets/diagrams/explicit-free-list/explicit_free_list-case-3.svg" alt="Case 3 - previous block allocated and next block free">
  <figcaption><em>Figure 11: Case 3 - previous block allocated and next block free</em></figcaption>
</figure>
<br><br>

**Case 4**: we coalesce the previous block, with the new and the next blocks. We remove the next block from the free list and update the next block pointer of the previous block
to correctly link to the new next block.
<figure id="figure-12">
  <img src="../../../public/assets/diagrams/explicit-free-list/explicit_free_list-case-4.svg" alt="Case 4 - previous and next blocks free">
  <figcaption><em>Figure 12: Case 4 - previous and next blocks free</em></figcaption>
</figure>
<br><br>


## Code base
And with that we have covered all the cases that we need to `malloc` and `free`. All the logic for `malloc` and `free` lives inside `mm.c`. My custom methods
are called `mm_malloc` and `mm_free`. To run the project
```bash
make
./test.out -d
```
The test driver includes a bunch of test cases and prints out the implicit and explicit free list states before and after each operation. If you want to see
how the initial heap is requested from the OS, check `memlib.c`. I use the `mmap` system call to obtain memory pages from the OS.

## Wrapping up
With explicit free lists you trade a bit of extra per-free-block overhead (the `prev`/`next` pointers) for much faster allocation
searches and easier removal/insert operations. This is a standard stepping stone on the way to production allocators.
Now we have a working heap allocator that we could use. But, to get to a production grade allocator like `malloc` from glibc there is still some more conceptual abstraction that we need to understand.
In the next article I’ll bridge the gap from this explicit free-list allocator to how a real `malloc` (e.g. glibc)
organizes bins, uses segregated lists, and optimizes splits/coalesces and large-allocation handling.
# Resources
* CS:APP - https://www.cs.sfu.ca/~ashriram/Courses/CS295/assets/books/CSAPP_2016.pdf
* Azeria Labs - https://azeria-labs.com/heap-exploitation-part-1-understanding-the-glibc-heap-implementation/


