% Destructors

What the language *does* provide is full-blown automatic destructors through the `Drop` trait,
which provides the following method:

```rust
fn drop(&mut self);
```

This method gives the type time to somehow finish what it was doing. **After `drop` is run,
Rust will recursively try to drop all of the fields of `self`**. This is a
convenience feature so that you don't have to write "destructor boilerplate" to drop
children. If a struct has no special logic for being dropped other than dropping its
children, then it means `Drop` doesn't need to be implemented at all!

**There is no stable way to prevent this behaviour in Rust 1.0**.

Note that taking `&mut self` means that even if you *could* suppress recursive Drop,
Rust will prevent you from e.g. moving fields out of self. For most types, this
is totally fine.

For instance, a custom implementation of `Box` might write `Drop` like this:

```rust
struct Box<T>{ ptr: *mut T }

impl<T> Drop for Box<T> {
	fn drop(&mut self) {
		unsafe {
			(*self.ptr).drop();
			heap::deallocate(self.ptr);
		}
	}
}
```

and this works fine because when Rust goes to drop the `ptr` field it just sees a *mut that
has no actual `Drop` implementation. Similarly nothing can use-after-free the `ptr` because
the Box is immediately marked as uninitialized.

However this wouldn't work:

```rust
struct Box<T>{ ptr: *mut T }

impl<T> Drop for Box<T> {
	fn drop(&mut self) {
		unsafe {
			(*self.ptr).drop();
			heap::deallocate(self.ptr);
		}
	}
}

struct SuperBox<T> { box: Box<T> }

impl<T> Drop for SuperBox<T> {
	fn drop(&mut self) {
		unsafe {
			// Hyper-optimized: deallocate the box's contents for it
			// without `drop`ing the contents
			heap::deallocate(self.box.ptr);
		}
	}
}
```

After we deallocate the `box`'s ptr in SuperBox's destructor, Rust will
happily proceed to tell the box to Drop itself and everything will blow up with
use-after-frees and double-frees.

Note that the recursive drop behaviour applies to *all* structs and enums
regardless of whether they implement Drop. Therefore something like

```rust
struct Boxy<T> {
	data1: Box<T>,
	data2: Box<T>,
	info: u32,
}
```

will have its data1 and data2's fields destructors whenever it "would" be
dropped, even though it itself doesn't implement Drop. We say that such a type
*needs Drop*, even though it is not itself Drop.

Similarly,

```rust
enum Link {
	Next(Box<Link>),
	None,
}
```

will have its inner Box field dropped *if and only if* an instance stores the Next variant.

In general this works really nice because you don't need to worry about adding/removing
drops when you refactor your data layout. Still there's certainly many valid usecases for
needing to do trickier things with destructors.

The classic safe solution to overriding recursive drop and allowing moving out
of Self during `drop` is to use an Option:

```rust
struct Box<T>{ ptr: *mut T }

impl<T> Drop for Box<T> {
	fn drop(&mut self) {
		unsafe {
			(*self.ptr).drop();
			heap::deallocate(self.ptr);
		}
	}
}

struct SuperBox<T> { box: Option<Box<T>> }

impl<T> Drop for SuperBox<T> {
	fn drop(&mut self) {
		unsafe {
			// Hyper-optimized: deallocate the box's contents for it
			// without `drop`ing the contents. Need to set the `box`
			// field as `None` to prevent Rust from trying to Drop it.
			heap::deallocate(self.box.take().unwrap().ptr);
		}
	}
}
```

However this has fairly odd semantics: you're saying that a field that *should* always
be Some may be None, just because that happens in the destructor. Of course this
conversely makes a lot of sense: you can call arbitrary methods on self during
the destructor, and this should prevent you from ever doing so after deinitializing
the field. Not that it will prevent you from producing any other
arbitrarily invalid state in there.

On balance this is an ok choice. Certainly what you should reach for by default.
However, in the future we expect there to be a first-class way to announce that
a field shouldn't be automatically dropped.