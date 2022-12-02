# Different approach for implementing a borrow checker
Today I introduce a method to implement a kind of borrow checker like rust's one that is more convenient for developers to use.
The idea is to associate references to actual referenced variables instead of scopes.
I'll call it references annotations
For instance:

    reference: &'a i32, // a reference to variable called a whose type is i32
Note that the `'a` may not be the real name of the variable but instead a denoted name, or even may mean multiple variables!
Also keep in mind that while the syntax may look too verbose at some points, most of the time the developer will not have to type annotations and when he has to it is mostly the basic ones.

Before continue reading be informed that while the syntax here is similar to that of rust's one, these concepts are not limited to a particular syntax, and I am not proposing to add anything to rust!

## Who refers to whom
Since the compiler can see and control all reference creation in a block of code it can determine which reference refers to which variable:

    struct Struct {
        field1: i32,
        field2: i32,
    }
    
    fn main() {
        let i1 = 0; // i1 is created
        let mut r = &i1; // r points to i1
        println!("{}", *r); // ok: r is valid because i1 is valid
        {
            let i2 = 0; // i2 is created
            r = &i2; // r points to i2
            println!("{}", *r); // ok: r is valid because i2 is valid
            // i2 is dropped here leaving r dangling
        }
        // println!("{}", *r); // error: r is invalid
        r = &i1; // r points to i1 again
        println!("{}", *r); // ok: r is valid because i1 is valid
        
        let s = Struct{ field1: 0, field2: 1 };
        r = &s.field1; // r points to s.field1
        println!("{}", *r); // ok: r is valid because s.field1 is valid
        r = &s.field2; // r points to s.field2
        println!("{}", *r); // ok: r is valid because s.field2 is valid
        drop(s); // s is dropped with all its fields leaving r dangling
        // println!("{}", *r); // error: r is invalid
    }

The compiler did all the job, and we didn't have to write any annotations. But what if we want to call another function and deal with references in parameters and return values?

## Functions
The compiler has two options to do the borrow checking when calling functions is involved:
- The compiler can step into the body of the function and continue analysing as if it was an inline block of code. This is currently used for lambdas, but not for ordinary functions since compile time will explode
- The compiler will determine what to do based on the function signature. So, we must encode the set of rules for the compiler to follow in the function signature.

Now how to do this?
Suppose we want to write a function that takes a reference pointing to some variable and returns a reference to the same variable, then we may write this:

    fn same_ref<'a>(r: &'a i32) -> &'a i32 {
        r
    }

Let's know how to interpret this:
- The function `same_ref<'a>` names a variable `'a` that resides outside the function body, but where is exactly that variable does not matter inside the function
- The parameter `r: &'a i32` is a reference to the variable `'a` whose type is `i32`
- The return type `&'a i32` is a reference to the same variable `'a`
- Inside the function body the compiler will make sure that the returned value `r` is pointing to `'a` as promised by the function signature

All the caller needs to use the function is its signature `fn same_ref<'a>(r: &'a i32) -> &'a i32` so we can write:

    let i1 = 0; // i1 is created
    // i1 is 'a and the resulting reference points to 'a which is i1!
    let r = same_ref(&i1); // r points to i1

Let's write a function that takes two references and returns either of them:

    fn max_of<'a, 'b>(r1: &'a i32, r2: &'b i32) -> &'(a | b) i32 {
        if *r1 >= *r2 {
            r1
        } else {
            r2
        }
    }

The function is more verbose than the previous one, but it isn't hard to understand:
- The function `max_of<'a, 'b>` names variables `'a` and `'b`  that reside outside the function body. Both `'a` and `'b` can point to the same variable or to different variables. More on this later
- The parameter `r1: &'a i32` points to `'a` and `r2: &'b i32` points to `'b`
- The returned reference `&'(a | b) i32` may point to either of `'a` or `'b`
- Inside the function body the compiler will make sure that the returned value `r1` or `r2` is pointing to either `'a` or `'b` as promised by the function signature

Here how this function may be used:
   

        let i1 = 0; // i1 is created
        {
            let i2 = 1; // i2 is created
            // i1 is 'a and i2 is 'b, the resulting reference points to either 'a or 'b so either i1 or i2! 
            let r = max_of(&i1, &i2); // r points to either i1 or i2
            // i2 is dropped here leaving r dangling
        }
Wait! can one reference `r` point to two variables? Really no. But since the compiler does not know which of the two variable the reference is pointing to it will be conservative and assume it points to either of them. This is why I said in the begin that one annotation `'a` may refer to a set of variables not only one.

The good news is that most of the references used will be tied to only one variable and when tied to many variables the set is limited and the compiler should know the set of variables that a reference may point to, but it can't deduce which one is actually pointed to

### Out references
There is another method a function can return references with, a kind of out references where you pass the function a reference to reference or a reference to a structure that has a reference field, and the function will make this inner reference point to another one
Consider the following function:

    fn move_a_to_b<'a, 'b>(r1: &mut &'a i32, r2: &mut &'b i32) {
        // the inner reference of r1 points to 'a
        // // the inner reference of r2 points to 'b
        *r1 = *r2; // the inner reference of r1 points to the inner of r2 thus points to 'b now
    }
notice how we didn't add annotations to `r1` and `r2` because we don't care about what does they refer to, only we are interested in the inner references pointing to `'a` and `'b` respectively.
The caller will call the function like this:

    let i1 = 0; // i1 is created
    let i2 = 1; // i2 is created
    let mut r1 = &i1; // r1 points to i1
    let mut r2 = &i2; // r2 points to i2
    move_a_to_b(&mut r1, &mut r2); // error: how can the compiler know which variable is pointed to by each reference?
If you try to compile this with rust compiler you will get:
```
error: lifetime may not live long enough
*r1 = *r2; // the inner reference of r1 points to the inner of r2 thus points to 'b now
   |     ^^^^^^^^^ assignment requires that `'b` must outlive `'a`
= help: consider adding the following bound: `'b: 'a`
```
The rust compiler tries to figure out the lifetime of references, but it can't prove that `'b` outlives `'a` so it complains in the assignment line and ask to add a bound `'b: 'a`
On the other hand, the introduced borrow checker will let the function compile fine since it is not the responsibility of the function to deduce the lifetime of its parameters, instead it should tell the compiler how it deals with references and since its signature does not tell anything about this, the compiler will complain in the call site saying that it can't match references to variables anymore.
The question now is how to encode this operation in the function signature?
Rust already supports two methods: bounds in the annotations declarations and bounds with `where` clause. 

Here I'll take another approach. I'll say a function may have preconditions and/or postconditions, both described using `where` clause followed by first block for preconditions (may be empty) and optional second block for postconditions.
The full syntax is:

    where { preconditions } { postconditons }
Again, be informed that syntax used here is for explaining and description only and I don't propose any syntax for any programming language, and a good programming language designer will mostly find a nicer syntax.

Let's keep the preconditions block empty for now and go to postconditions.
To say that after the function returns `r1` will be pointing to `'b` add this to postconditions:

    r1: &mut &'b i32
The function becomes:

    fn move_a_to_b<'a, 'b>(r1: &mut &'a i32, r2: &mut &'b i32) where {} { r1: &mut &'b i32 } {
        // the inner reference of r1 points to 'a
        // // the inner reference of r2 points to 'b
        *r1 = *r2; // the inner reference of r1 points to the inner of r2 thus points to 'b now
    }
the compiler now can use the function signature to compile this code:
   

        let i1 = 0; // i1 is created
        let i2 = 1; // i2 is created
        let mut r1 = &i1; // r1 points to i1
        let mut r2 = &i2; // r2 points to i2
        // i1 is 'a and i2 is 'b
        move_a_to_b(&mut r1, &mut r2); 
        // r1 points now to i2 ('b), and r2 is unchanged

If the function will not change `r1` in all paths it may use the postcondition `r1: &mut &'(a + b) i32` instead and the compiler will assume that `r1` points to either of them after return

The syntax got too verbose, right? Yes, but you don't need to define eached annotation and dealing with out references like this is rare unlike return types.
For example, you can omit `'a` from the previous function like any reference annotation you don't use in return types and don't assign to another reference in the parameters.
There are also rules for the compiler to put the annotations for you in general cases similar to rust's rules.

## Structs
A structure generally contains number of fields that are constructed together in a specified order and dropped together in a specified order. Each member in a struct can be referenced individually and a reference to one field must not be mistaken to another one and is not the same as a reference to the struct itself! This point is important and differs from rust's borrow checker's rules.

    struct Struct {
        field1: i32,
        field2: i32,
    }
    let s = Struct{ field1: 0, field2: 1 };
    let r1 = &s.field1; // r1 points to s.field1
    println!("{}", *r1); // ok: r1 is valid because s.field1 is valid
    let r2 = &s.field2; // r2 points to s.field2
    println!("{}", *r2); // ok: r2 is valid because s.field2 is valid

Rust already differentiates between direct fields borrows as above but the situation is different when returning fields from a function:

    fn get_field1<'a>(s: &'a Struct) -> &'a i32 {
        &s.field1
    }
This can't be done in the introduced borrow checker because:
- The compiler needs to know which object the reference points to, and the function doesn't tell which field of the struct is returned by reference
- In the function signature we named a variable `'a` with type `Struct` in the parameter but it appeared at the return with type `i32`!

So, a new syntax is required to get this code work.
Recall that `r: &'a i32` means that reference `r` points to variable `'a` of type `i32` and `a` may be the real name of the variable or introduced by function, struct, impl, ...
In order to tie a reference to field of struct the annotation will be the real name of the field just as you are using it:

    fn get_field1(s: &Struct) -> &'s.field1 i32 {
        &s.field1
    }
Or by field index:

    fn get_field1(s: &Struct) -> &'s.0 i32 {
        &s.field1
    }
For associated methods which start with `self` parameter the field name or index may be used directly:

    impl Struct {
        fn get_field1(&self) -> &'self.0 i32 {
            &self.field1
        }
        
        fn get_field2(&self) -> &'1 i32 {
            &self.field2
        }
    }
nested fields annotations are written in the same manner, for example: `&'s.0.1`
Why one needs the type of field while the field path is typed in the annotation?
Because the function may not return a reference to a  `dyn` trait that is implemented by the field. In other cases, the type may be omitted but it is not a big deal anyway.

### Reference fields in structs
A struct may contain a reference as a field so there must be a syntax to tie this reference to its pointed to variable. The rust syntax can just do it with minor changes:

    // 'a and 'b names two set of variables which may or may not be the same
    struct TwoRefs<'a, 'b> {
        r1: &'a i32, // a is i32
        r2: &'b i32, // b is i32
        r3: &'a i32, // r3 is always pointing to the same variable pointed to by r1
        //r4: &'a f64, // error a can't be i32 and f64
        r5: &'a dyn Trait, // ok: a may be referenced as Trait since Trait is implemented for i32
        r6: &('a | 'b) i32, // r6 points to either 'a or 'b, assume it points to both
    }

Inside the struct `impl` block reference parameters of functions that points to the same struct variables sets `'a` and `'b` are added to the set because the function may not assign the reference in all paths If it is required to replace the variables set instead of adding to it, one may use `''a` instead of `'a` and this will be enforced in the function body

    impl<'a, 'b> TwoRefs<'a, 'b> {
        // the resulting struct points to 'a and 'b
        fn new(r1: &'a i32, r2: &'b i32) -> Self {
            Self {
                r1: r1,
                r2: r2,
                r3: r1,
                r5: r1,
                r6: r2, // or r6: r1
            }
        }
        
        // add 'a variable referenced by r to the struct's 'a set
        fn add_to_a(&mut self, r: &'a i32) {
            if *r1 > 0 {
                r1 = r;
            }
            r5 = r;
        }
        
        // set the 'b set to ''b referenced by r
        fn_set_b(&mut self, r: &''b i32) {
            // must set all references that may point to 'b to r 
            r2 = r;
            r6 = r;
        }
    }
    
    fn get_a<'a>(t: &TwoRefs<'a, '_>) -> &'a i32 {
        //t.r6 // error: r6 points to either 'a or 'b but return must point to 'a
        //t.r3 // ok
        t.r1
    }
    
    fn main() {
        let i1 = 0;
        let i2 = 1;
        let mut t = TwoRefs::new(&i1, &i2);
        // t.r1 and t.r3 and t.r5 and t.r6 points to i1
        // t.r2 and t.r6 points to i2
        
        let i3 = 2;
        t.add_to_a(&i3); // .r1 and t.r3 and t.r5 and t.r6 points to both i1 and i3
        
        let i4 = 3;
        t.set_b(&i4); // t.r2 and t.r6 points to i4, t.r6 no longer points to i3
        
        let r = get_a(&t); // r points to both i1 and i3
    }

## Multiple mutable references
This is a known limitation in rust, that is: it does not support multiple `mut` references to the same object or mixed `mut` and non `mut` references, instead you need to use `RefCell` and `Cell` and let the safety checks be done in runtime. However, you can't just plug in `RefCell` and go since these wrappers don't go well with other types. Either you wrap all your type in cells, or you will find yourself always moving your variables to and from a cell.
Another caveat is that the cells can be mutated without being declared `mut` so it goes beyond mutable by default to no `const` in the language!
Not to mention that some entire architectures use mutably shared data extensively like retained GUI and MVC programming (Model View Controller). This explains why rust does not have until now usable retained GUI library and each developer is coming with a new architecture that comes nowhere near the proved and established ones.  

But why rust did go this way? The problem is that some wrapper types like `enum` , `Box`, `Vec` and other pointers and containers may return you a reference to a value that is allocated on the heap and if the container for example clears its storage the returned reference will be invalid! The same applies to iterators. This is known among `c++` developers like iterator invalidation and use after move.

Look at this similar struct to `Box` introduced in rust docs:

    struct Carton<T>(ptr::NonNull<T>);
    
    impl<T> Carton<T> {
        pub fn new(value: T) -> Self {
            // Allocate enough memory on the heap to store one T.
            assert_ne!(size_of::<T>(), 0, "Zero-sized types are out of the scope of this example");
            let mut memptr: *mut T = ptr::null_mut();
            unsafe {
                let ret = libc::posix_memalign(
                    (&mut memptr).cast(),
                    align_of::<T>(),
                    size_of::<T>()
                );
                assert_eq!(ret, 0, "Failed to allocate or invalid alignment");
            };
    
            // NonNull is just a wrapper that enforces that the pointer isn't null.
            let ptr = {
                // Safety: memptr is dereferenceable because we created it from a
                // reference and have exclusive access.
                ptr::NonNull::new(memptr)
                    .expect("Guaranteed non-null if posix_memalign returns 0")
            };
    
            // Move value from the stack to the location we allocated on the heap.
            unsafe {
                // Safety: If non-null, posix_memalign gives us a ptr that is valid
                // for writes and properly aligned.
                ptr.as_ptr().write(value);
            }
    
            Self(ptr)
        }
    }
    
    impl<T> Deref for Carton<T> {
        type Target = T;
    
        fn deref(&self) -> &Self::Target {
            unsafe {
                // Safety: The pointer is aligned, initialized, and dereferenceable
                //   by the logic in [`Self::new`]. We require readers to borrow the
                //   Carton, and the lifetime of the return value is elided to the
                //   lifetime of the input. This means the borrow checker will
                //   enforce that no one can mutate the contents of the Carton until
                //   the reference returned is dropped.
                self.0.as_ref()
            }
        }
    }
    
    impl<T> DerefMut for Carton<T> {
        fn deref_mut(&mut self) -> &mut Self::Target {
            unsafe {
                // Safety: The pointer is aligned, initialized, and dereferenceable
                //   by the logic in [`Self::new`]. We require writers to mutably
                //   borrow the Carton, and the lifetime of the return value is
                //   elided to the lifetime of the input. This means the borrow
                //   checker will enforce that no one else can access the contents
                //   of the Carton until the mutable reference returned is dropped.
                self.0.as_mut()
            }
        }
    }
    
	impl<T> Drop for Carton<T> {
        fn drop(&mut self) {
            unsafe {
                libc::free(self.0.as_ptr().cast());
            }
        }
    }

Imagine that the compiler will allow you to get multiple `mut` references to `Carton` and write the following code:

        let mut c = Carton::new(0); // c is Carton<i32> holding i32 value allocated on the heap
        let r: &mut i32 = &mut c; // r is &i32 borrowing c
        c = Carton::new(0); // assign c to a new Carton<i32>, the old storage is freed leaving r dangling
        println!("{}", *r); // error: use after free, r is dangling!
The code clearly invokes a bug, and rut prevents it by preventing the use of `mut c` while `r` is still in use

How can this be worked around?
Let's introduce a new special type called `PhantomMut<T>` . This is not the best name to call it, but it suffices for now.
This type, like rust's phantom types, does not have storage in the containing `struct` . The special thing about `PhantomMut<T>` is that it is considered dropped after each mutation of its struct and a new `PhantomMut<T>` is created so references to this type gets invalidated each time this type is mutated.
How does this help us build a `Box`?

    struct Carton<T> {
        p: ptr::NonNull<T>,
        pub _0: PhantomMut<T>,
    }
    
    impl<T> Deref for Carton<T> {
        type Target = T;
    
        // remeber that references to fields are not references to the struct
        fn deref(&self) -> &'_0 Self::Target {
            unsafe {
                // unsafe allows converting reference with unknown variable to &'_0
                self.p.as_ref()
            }
        }
    }
    
    impl<T> DerefMut for Carton<T> {
        // after this method all previous references to '_0 are invalid
        fn deref_mut(&mut self) -> &'_0 mut Self::Target {
            unsafe {
                // unsafe allows converting reference with unknown variable to &'_0
                self.0.as_mut()
            }
        }
    }
    
    impl<T> Drop for Carton<T> {
        // after this method all previous references to '_0 are invalid
        fn drop(&mut self) {
            unsafe {
                libc::free(self.0.as_ptr().cast());
            }
        }
    }

Let's use it

    let mut c = Carton::new(0); // c is Carton<i32> holding i32 value allocated on the heap
    let r: &mut i32 = &mut c; // r is &i32 borrowing c._0
    c = Carton::new(0); // assign c to a new Carton<i32>, drop is called, c._0 is reset, the old storage is freed and r is invalid
    //println!("{}", *r); // error: use after free, r is dangling!
    r = &mut c; // r is valid again

The compiler now can allow a mutable reference to `c` and `c._0` and will detect when the reference to `c._0` becomes invalid.
But you will notice that the example above is somewhat useless because if we try to dereference `Carton` multiple times the previous references will be invalid but the pointed to value is still valid! So, we can't get more than a `mut` reference to the value of the box stored on the heap.
To overcome this, we will use a new postcondition to tell the compiler not to reset `c._0` after the call to `fn deref_mut(&mut self) -> &'_0 mut Self::Target`.

    impl<T> DerefMut for Carton<T> {
        // after this method all previous references to '_0 are invalid
        fn deref_mut(&mut self) -> &'_0 mut Self::Target where {} { _0: const } {
            unsafe {
                // unsafe allows converting reference with unknown variable to &'_0
                self.0.as_mut()
            }
        }
    }

We can now call `deref_mut` multiple times and all the references pointing to `c._0` will still be valid, only on drop they are invalidated.

The same applies to `Vec`: `index` and `index_mut` will return a reference pointing to `vec._0` and operations like `push`, `pop`, `clear`... will invalidate the references and iterators to the data allocated by the vector.

## Enums (tagged unions)
In rust tagged union are backed in the language under the name of `enum` (not to be confused with `c`/`c++` `enum`)
If current rust's borrow checker allowed multiple `mut` references to an `enum` references may be dangling without a way to detect that and use after free bugs may arise.
I took this example from a blog on the web:

    enum StringOrInt {
        Str(String),
        Int(i64)
    }
    
    let x = Str("Hi!".to_string()); // Create an instance of the `Str` variant with associated string "Hi!"
    let y = &mut x; // Create a mutable alias to x
    
    if let Str(ref insides) = x { // If x is a `Str`, assign its inner data to the variable `insides`
        *y = Int(1); // Set `*y` to `Int(1), therefore setting `x` to `Int(1)` too
        println!("x says: {}", insides); // Uh oh!
    }
By making all `enum` types have a public field `_0: PhantomMut<T1>`, `_1: PhantomMut<T2>`, ... for each type `T` in the `enum` all references referring to a type in the `enum` will be pointing to that field and when the `enum` is dropped or assigned all public fields are reset invalidating references, except the one associated with the type being set

    enum StringOrInt {
        Str(String),
        Int(i64),
    	// pub _0: PhantomMut<Str>,
    	// pub _1: PhantomMut<Int>,
    }
    
    let x = Str("Str".to_string()); // Create an instance of the `Str` variant
    let y = &mut x; // Create a mutable alias to x
    
    if let Str(ref insides) = x { // insides is a field inside Str variant which is &'x._0 Str
        *y = Int(1); // Set `*y` to `Int(1), therefore setting `x` to `Int(1)` too, therefore x._0 is reset leaving &'x._0 Str dangling, and all its fields are invalid
        //println!("x says: {}", insides); // error: insides is invalid after x._0 was reset
    }
## Arrays
arrays can be thought as a fixed size `Vec` at which all elements are constructed together in a specified order, and dropped together in a specified order.
An array of type `[T;N]` will have a public field `_0: PhantomMut<T>` which all references to elements of the array are tied to, and when the array is dropped all references to its elements are invalidated

### Swap implementation with multiple mutable aliases
Rust has a swap function that does bitwise swap of two variables of the same type

    fn swap<T>(x: &mut T, y: &mut T)
the usual implementation is to use `memcpy` to copy the content of `x` to a temporary place, then copy content of `y` to `x` and retore `x` contents from temp into `y`. However, an implementation may do some optimizations based on the assumption that `x` and `y` does not alias. With multiple mutable aliases allowed how a function can assert that two mutable references in its parameters don't alias?
Using function preconditions:

    fn swap<T, 'a, 'b>(x: &'a mut T, y: &'b mut T) where { 'a != 'b } { /* function body */ }
Without this precondition the compiler can't deduce that inside the function body `'a` and `'b` do not alias so it will be conservative and assume that all references of the same type passed in parameters may point to one variable and make analysis based on this. This will lead us to the next section, but before going to the next section, how can we swap two parameters if we for some reason couldn't provide a precondition to require that they don't alias? Specialize the swap function:

    fn swap<T>(x: &mut T, y: &mut T) { /*  assume x and y refer to the same memory and check if address of x equals that of y first */ }
 In fact, this is used in `c++` assignment operators nearly everywhere because mutable references may alias each other. The situation with a borrow checker is better since the compiler can easily use the best specialization based on the variables it is passing to the function
 ### Assume all references in parameter are invalid after one reference of the same type is invalid, unless proved the opposite
 If we have this function:

    fn mutate_carton(c: &mut Carton<i32>, i: &i32) {
        c = Carton::new(0); // drop and assign c to a new Carton, reset c._0 and invalidate any pointer to c._0
        *i = 1; // what if i was referring to c._0 ? a potential use after free bug!
    }
    
and it used like this:

    let mut c = Carton::new(0);
    mutate_carton(&mut c, &c);

This will lead to a subtle bug based on what input is given to the function. The current rust borrow checker doesn't allow this code to begin with. But with multiple mutable aliases allowed, this code may compile and cause an after free bug.
To mitigate this problem, when the compiler attempts to invalidate a reference of type `T` in the parameters of a function, it will invalidate all references with the same type `T` or a trait implemented by `T` in the parameters, unless the compiler can prove (with aid of precondition) that a particular reference does not alias with other references. So, to requires that `i` does not point to  `c._0` we may write:

    fn mutate_carton<'a>(c: &mut Carton<i32>, i: &'a i32) where { 'a != 'c._0 } {
        c = Carton::new(0); // drop and assign c to a new Carton, reset c._0 and invalidate any pointer to c._0
        *i = 1; // i does not point to c._0 so it is still valid
    }
The following code will not compile because it violates the function preconditions:

    let mut c = Carton::new(0);
    mutate_carton(&mut c, &c); // error c points to c._0

How does the compiler know it is invalidating references to `i32` when `c._0` is reset? It's because `c._0` has a type of `PhantomMut<i32>`, and this is why `PhantomMut<T>` is generic and why the compiler must know the type of each reference annotation in a struct to know which reference to invalidate and which to not.

## Static variables and references
All static variables are accessible by all threads and are either immutable or `mut` and `Sync` where `Sync` means it can be mutated from multiple threads (like a mutex or atomic type) or it is immutable and thus can be read by multiple threads
Static variables also last from before the main function begins until after it returns so it is always valid to access after the main function was entered.
Generally speaking, rust does not allow to do advanced things with static variables like in `c++` in a safe manner so we will not talk about order of initialization and related problems.
From a function point of view, if a reference points to a static variable `r: &'static i32` it is not very helpful to know which static variable actually is referenced by the reference, and if a function returns a static reference the caller does not need to know which static variable is referenced.

A more useful thing for generic programming, instead of relying on `'static` to require that a type is static a trait `Static` is more useful. This trait will be auto implemented for types and structs that does not contain reference, references to static variables and structs that depend only on static variables. This should make it easier to specialize generic functions for `Static` variables but implementing this trait for references and structs that depend only on static variables is tricky! It's because a reference that might now point to a static variable might not point to it later, and a struct that depends on a static variable may not depend on it later, leaving a burden on the borrow checker to implement and un implement a trait for type. This is clearly not the job of the borrow checker and this clarifies why rust specialization has not reached stabilization yet!

## What about threads? 
As far as I know, there exists two kinds of safe thread types:
- Detached thread (start and forget) which requires its functor to be static and since static variables are `Sync` or immutable, detached thread should just work as it did before
- Scoped thread where you have spawn threads and block the current thread until they have finished executing so spawned threads can refer to data in the local thread stack and since multiple mutable references can alias, a data race may occur, right?

Examine this example from rust docs:

    let mut a = vec![1, 2, 3];
    let mut x = 0;
    
    thread::scope(|s| {
        s.spawn(|| {
            println!("hello from the first scoped thread");
            // We can borrow `a` here.
            dbg!(&a);
        });
        s.spawn(|| {
            println!("hello from the second scoped thread");
            // We can even mutably borrow `x` here,
            // because no other threads are using it.
            x += a[0] + a[2];
        });
        println!("hello from the main thread");
    });
    
    // After the scope, we can modify and access our variables again:
    a.push(4);
    assert_eq!(x, a.len());

Notes:
- threads eagerly starts with `s.spawn` 
- threads functors are required to be `Send`
- spawned threads can capture multiple immutable references
- only one `mut` reference to an object is allowed inside the scope block

This clearly won't work out of the box with multiple `mut` references allowed.
But it can be made to work through:
- threads are lazy started inside `thread::scope` after all threads are scheduled for spawning using `s.spawn`
- the functor of the thread muse be `Send` and `Sync` meaning that all references must either be immutable or safe to mutate from multiple threads

Pros: The current thread may participate in executing the spawned threads saving the cost of launching a new thread instead of just blocking
Cons: No one thread can capture a mutable reference to a local variable except if it is `Sync` so borrowing `x` as above won't work. This is not much of importance because it is more appropriate to mutate data on the current thread instead of sending a `mut` reference to them to another thread and then just block and wait for it to finish

## Things I didn't address or forgot
There are many aspects I didn't address, refer to or didn't know of or otherwise forgot to mention so I'm not surprised if any of the methods introduced here are wrong or make nonsense. Any feedback will be welcome!
