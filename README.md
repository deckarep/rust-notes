# rust-notes
A place for my rust notes. This is a repo that I'll be building out to aggregate my adventure with learning rust.  It will contain my own content as well as content from others and all Copyright Claim is held accordingly by each original author's works.

## [Intro to Rust - Alex Crichton](https://www.youtube.com/watch?v=agzf6ftEsLU)

### Axioms of Rust

#### Ownership and Borrowing
- There is only ever **one owner of data**
- Ownership can be transferred to a new owner; this is known as a **move**.
- Ownership is a **deep property** of a type
- Owned values can be **borrowed temporarily**
- Borrowed values are only valid for a **particular lifetime**
- **Borrowing prevents moving** -- while there is an active borrow, the owner cannot be moved
- Borrows can be nested
- Borrowed values can become owned values through **cloning**

#### Memory Management
- Each variable has **a scope it is valid for**, and it is **automatically deallocated** when it goes out of scope
- Reference counting is another way of managing memory -- RC type
- Rust has shared memory but you must **explicitly opt into it**

#### Mutability
- Values are **immutable by default**
- Mutability is also part of the type of a borrowed pointer
- Borrowed pointers may coerce
- Values can be **frozen by borrowing**
- **Mutability propagates deeply** into owned types (just like ownership does)

#### Concurrency
- Rust's channels enforce thread isolation
- Thread safety isn't just documentation; it's the law
- Even the most daring forms of sharing are guaranteed safe in Rust
- Lock data; not code is enforced in Rust


## Iterators [Rust Camp 2015 - Who Owns this Stream of Data?](https://www.youtube.com/watch?v=NGW17shYtRM)

### IntoIter - Owned Values (T)
- Moves the data out of the collection
- You get total ownership
- Can do anything with the data, including destroy it

```Rust
fn process(data: Vec<String>){
	for s in data.into_iter() {
		println!("{}", s);
	}

    // Oh no! Iterating consumed it :(
	println!("{}", data.len()); //~ERROR
}
```

### Iter - Shared References (&T)
- Iter lets you look but not touch :)
- Shares the data in the collection
- Read-only access
- Can have many readers at once

```Rust
fn print(data: &Vec<String>){
	for s in data.iter(){
		//All I can do is read :/
		println!("{}", s);
	}
    // Yay it lives!
	println!("{}", data.len());
}
```

### IterMut - Mutable References (&mut T)
- Loans the data in the collection
- Read-Write access
- Only one loan at once
- IterMut gives you exclusive access

```Rust
fn make_better(data: &mut Vec<String>){
	for s in data.iter_mut(){
		//Ooh I can mutate you!
		s.push_str("!!!!");
		//But I can't share :(
	}
}
```

## [LamdaConf 2015 - In Rust we Trust Alex Burkhart](https://www.youtube.com/watch?v=-dxqbhLIgdM)

### Data Race
- 2 + threads accessing the same data
- at least 1 is unsynchronized
- at least 1 is writing

### Shared Nothing

```Rust
use std::sync::mpsc::channel;
use std::thread;

fn main() {
    let (tx, rx) = channel();

    for task_num in 0..8 {
        let tx = tx.clone();
        thread::spawn(move || {
            let msg = format!("Task {:?} done!", task_num);
            tx.send(msg).unwrap();
        });
    }

    drop(tx); // Effectively closes the channel

    for data in rx {
        println!("{:?}", data);
    }
}
```

### Shared Immutable Memory

```Rust
use std::sync::mpsc::channel;
use std::thread;
use std::sync::Arc;

struct HugeStruct{
   name: String
}

fn main() {
    let (tx, rx) = channel();
    let huge_struct = HugeStruct{name:"Ralph".into()};
    let arc = Arc::new(huge_struct);

    for task_num in 0..8 {
        let tx = tx.clone();
        let arc = arc.clone();
        thread::spawn(move || {
            let msg = format!("Task {:?} Accessed {:?}", task_num, arc.name);
            tx.send(msg).unwrap();
        });
    }

    drop(tx); // Effectively closes the channel

    for data in rx {
        println!("{:?}", data);
    }
}
```

### Mutation with Synchronization
```Rust
use std::sync::mpsc::channel;
use std::thread;
use std::sync::Arc;
use std::sync::Mutex;

struct HugeStruct{
   name: String,
   access_count: i32
}

fn main() {
    let (tx, rx) = channel();
    let huge_struct = HugeStruct{name:"Ralph".into(), access_count:0};
    let arc = Arc::new(Mutex::new(huge_struct));

    for task_num in 0..8 {
        let tx = tx.clone();
        let arc = arc.clone();
        thread::spawn(move || {
            let mut guard = arc.lock().unwrap();
            guard.access_count +=1;
            let msg = format!("Task {:?} Accessed {:?} Name {:?}", task_num, guard.access_count, guard.name);
            tx.send(msg).unwrap();
        });
    }

    drop(tx); // Effectively closes the channel

    for data in rx {
        println!("{:?}", data);
    }
}
```

### [Fun Example: A generator function](http://stackoverflow.com/a/31392115/71079)
```Rust
use std::thread;
use std::sync::mpsc;

fn gen_range(start: i32, end: i32) -> mpsc::Receiver<i32> {
    let (tx, rx) = mpsc::channel::<i32>();
    thread::spawn(move || {
        for i in start..end {
            tx.send(i).unwrap();
        }
        drop(tx);
    });
    return rx;
}

fn main() {
    let range = gen_range(0, 50);
    
    for x in range {
        println!("{}", x);
    }
}
```

### Ranges, Arrays, Slices, Vectors, Iterators oh my...

#### [Ranges](http://killercup.github.io/trpl-ebook/trpl-2015-09-26.a4.pdf)
```Rust
 	// Key take-aways: 
    // - Ranges start off as iterators (notice no need to convert)
    // - Ranges can represent infinite sequences
    // - Collecting must occur due to lazy evaluation
    
    // This does not actually generate anything
    let nums = 1..100;
    
    // It must be consumed
    let nums = (1..100).collect::<Vec<i32>>();

    // Collect a range into a vector
    let one_to_one_hundred = (1..101).collect::<Vec<_>>();
    print_type_of(&one_to_one_hundred); //std::vec::Vec<i32>

    // Find numbers in range returning an Option
    let greater_than_forty_two = (0..100).find(|x| *x > 42);
    print_type_of(&greater_than_forty_two); //std::option::Option<i32>

    match greater_than_forty_two {
        Some(_) => println!("We got some numbers!"),
        None => println!("No numbers found :("),
    }
    
    // Filter numbers in range returning a vector
    let greater_than_forty_two = (0..100)
        .filter(|x| *x > 42)
        .collect::<Vec<_>>();

    print_type_of(&greater_than_forty_two); //std::vec::Vec<i32>
    println!("greater_than_forty_two: {:?}", greater_than_forty_two);
```


### [Let's get functional](https://mmstick.gitbooks.io/rust-programming-phoronix-reader-how-to/content/chapter02.html)

#### With numbers again
```Rust
let numbers_iterator = [0,2,3,4,5].iter();

let sum = numbers_iterator
 	.fold(0, |total, next| total + next)
	.collect();

let squared = (1..10).iter()
	.map(|&x| x * x).collect();
```

#### With numbers again
```Rust
fn main() {
    let inf_range = (1..)         // Infinite range of integers
        .filter(|x| x % 2 != 0)   // Collect odd numbers
        .take(5)                  // Only take five numbers
        .map(|x| x * x)           // Square each number
        .collect::<Vec<usize>>(); // Return as a new Vec<usize>

    println!("{:?}", inf_range);  // Print result
}
```

#### With Strings
```Rust
fn main() {
    let sentence = "This is a sentence in Rust.";
    let words: Vec<&str> = sentence
  		.split_whitespace()
        .collect();

    let words_containing_i: Vec<&str> = words.into_iter()
        .filter(|word| word.contains("i"))
        .collect();

    println!("{:?}", words_containing_i);
}
```

### What type am I?
This feature requires Rust Nightly, this helper method allows for figuring out what type you are working with
Just pass a reference to it of whatever type you are holding

```Rust
#![feature(core_intrinsics)]

fn print_type_of<T>(_: &T) -> () {
    let type_name = unsafe { std::intrinsics::type_name::<T>() };
    println!("{}", type_name);
}
```
