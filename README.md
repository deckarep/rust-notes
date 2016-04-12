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
- Parallelism is achieved at the granularity of an OS thread
- When using mutexes, lock data not code
- Safety is achieved by requiring that a `proc` owns captured variables  <== to to check this for newer Rust
- Threads can communicate with channels
- Tasks can also share memory -- Arc type <== check this for newer Rust


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
