# axalloc

Global allocator for kernel.
It provides `GlobalAllocator`, which implements the trait `core::alloc::GlobalAlloc`. A static global variable of type `GlobalAllocator` is defined with the `#[global_allocator]` attribute, to be registered as the standard library’s default allocator.

## Examples

```rust
#![no_std]
#![no_main]

#[macro_use]
extern crate axlog2;
extern crate alloc;
use alloc::string::String;

use core::panic::PanicInfo;

#[no_mangle]
pub extern "Rust" fn runtime_main(_cpu_id: usize, _dtb_pa: usize) {
    axlog2::init("debug");
    info!("[rt_axalloc]: ...");

    axalloc::init();

    let s = String::from("Hello, axalloc!");
    info!("Alloc string: {}", s);
    info!("[rt_axalloc]: ok!");

    axhal::misc::terminate();
}

#[panic_handler]
pub fn panic(info: &PanicInfo) -> ! {
    arch_boot::panic(info)
}
```

## Functions

### `global_add_memory`

```rust
pub fn global_add_memory(start_vaddr: usize, size: usize) -> AllocResult
```

Add the given memory region to the global allocator.
Users should ensure that the region is valid and not being used by others, so that the allocated memory is also valid.
It’s similar to global_init, but can be called multiple times.

### `global_allocator`

```rust
pub fn global_allocator() -> &'static GlobalAllocator
```

Returns the reference to the global allocator.

### `global_init`

```rust
pub fn global_init(start_vaddr: usize, size: usize)
```

Initializes the global allocator with the given memory region.
Note that the memory region bounds are just numbers, and the allocator does not actually access the region. Users should ensure that the region is valid and not being used by others, so that the allocated memory is also valid.
This function should be called only once, and before any allocation.

## Struct

### `GlobalAllocator`

```rust
pub struct GlobalAllocator {
    balloc: SpinNoIrq<DefaultByteAllocator>,
    palloc: SpinNoIrq<BitmapPageAllocator<PAGE_SIZE>>,
}
```

It combines a `ByteAllocator` and a `PageAllocator` into a simple two-level allocator: firstly tries allocate from the byte allocator, if there is no memory, asks the page allocator for more memory and adds it to the byte allocator.
Currently, `TlsfByteAllocator` is used as the byte allocator, while `BitmapPageAllocator` is used as the page allocator.
`TlsfByteAllocator`: allocator::TlsfByteAllocator

#### Implementations

##### `impl GlobalAllocator`

```rust
pub const fn new() -> Self
```

Creates an empty GlobalAllocator.

```rust
pub const fn name(&self) -> &'static str
```

Returns the name of the allocator.

```rust
pub fn init(&self, start_vaddr: usize, size: usize)
```

Initializes the allocator with the given region.
It firstly adds the whole region to the page allocator, then allocates a small region (32 KB) to initialize the byte allocator. Therefore, the given region must be larger than 32 KB.

```rust
pub fn add_memory(&self, start_vaddr: usize, size: usize) -> AllocResult
```

Add the given region to the allocator.
It will add the whole region to the byte allocator.

```rust
pub fn alloc(&self, layout: Layout) -> AllocResult<NonNull<u8>>
```

Allocate arbitrary number of bytes. Returns the left bound of the allocated region.
It firstly tries to allocate from the byte allocator. If there is no memory, it asks the page allocator for more memory and adds it to the byte allocator.

align_pow2 must be a power of 2, and the returned region bound will be aligned to it.

```rust
pub fn dealloc(&self, pos: NonNull<u8>, layout: Layout)
```

Gives back the allocated region to the byte allocator.
The region should be allocated by alloc, and align_pow2 should be the same as the one used in alloc. Otherwise, the behavior is undefined.

```rust
pub fn alloc_pages(
    &self,
    num_pages: usize,
    align_pow2: usize
) -> AllocResult<usize>
```

Allocates contiguous pages.
It allocates num_pages pages from the page allocator.
align_pow2 must be a power of 2, and the returned region bound will be aligned to it.

```rust
pub fn dealloc_pages(&self, pos: usize, num_pages: usize)
```

Gives back the allocated pages starts from pos to the page allocator.
The pages should be allocated by alloc_pages, and align_pow2 should be the same as the one used in alloc_pages. Otherwise, the behavior is undefined.

```rust
pub fn used_bytes(&self) -> usize
```

Returns the number of allocated bytes in the byte allocator.

```rust
pub fn available_bytes(&self) -> usize
```

Returns the number of available bytes in the byte allocator.

```rust
pub fn used_pages(&self) -> usize
```

Returns the number of allocated pages in the page allocator.

```rust
pub fn available_pages(&self) -> usize
```

Returns the number of available pages in the page allocator.

#### Trait Implementations

##### `impl GlobalAlloc for GlobalAllocator`

```rust
unsafe fn alloc(&self, layout: Layout) -> *mut u8
```

Allocate memory as described by the given layout. [Read more](https://doc.rust-lang.org/nightly/core/alloc/global/trait.GlobalAlloc.html#tymethod.alloc)

```rust
unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout)
```

Deallocate the block of memory at the given ptr pointer with the given layout. [Read more](https://doc.rust-lang.org/nightly/core/alloc/global/trait.GlobalAlloc.html#tymethod.alloc)

```rust
unsafe fn alloc_zeroed(&self, layout: Layout) -> *mut u8
```

Behaves like alloc, but also ensures that the contents are set to zero before being returned. Read more

```rust
unsafe fn realloc(
    &self,
    ptr: *mut u8,
    layout: Layout,
    new_size: usize
) -> *mut u8
```

Shrink or grow a block of memory to the given new_size in bytes. The block is described by the given ptr pointer and layout. [Read more](https://doc.rust-lang.org/nightly/core/alloc/global/trait.GlobalAlloc.html#method.realloc)
