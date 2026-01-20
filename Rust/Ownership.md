[The Rust Programming Language - 4. Ownership](https://doc.rust-lang.org/book/ch04-00-understanding-ownership.html)



# Ownership

**Rust Ownership 三大规则**：

1. **每个值都有一个所有者（owner）**
2. **值在任意时刻只能有一个所有者**
3. **当所有者离开作用域时，值将被丢弃（drop）**

```rust
let s1: String = String::from("hello");
let s2 = s1;
println!("{}", s1)		// error: borrow of moved value: `s1`
						// 此时 s1 的所有权移动到 s2，s1 已失效

let x1: i32 = 10;
let x2: i32 = x1;
println!("{}, {}", x1, x2)	// pass
							// i32 和 String 类型不一样，所有权转移有区别
```

> **Copy Trait**
>
> 由于 `i32` 实现了 Copy Trait 的类型在赋值时自动发生隐式复制，不转移所有权，原变量 **保持有效**。但是 `String` 并没有。
>
> 实现了 Copy 的类型：
>
> - 所有整数类型：`i8`-`i128`, `u8`-`u128`, `isize`, `usize`
> - 浮点类型：`f32`, `f64`
> - 布尔：`bool`
> - 字符：`char`
> - 不可变引用：`&T`
> - 函数指针：`fn()`
> - 元组（所有成员都是 Copy）
> - 数组（元素是 Copy，固定大小）
>
> **不能实现 Copy**：
>
> - `String`, `Vec<T>`, `Box<T>`（堆分配）
> - `&mut T`（可变引用）
> - 实现了 `Drop` 的类型
> - 包含非 Copy 字段的结构体

权限定义在位置上，位置是任何可以放在赋值语句左侧的东西。



# References and Borrowing

> We call the action of creating a reference *borrowing*.

***“引用遵循『编译期读写锁（RWLock）』原则：同一时刻，要么有任意数量的读者（&T），要么只有唯一的写者（&mut T），二者绝对互斥且写者独占。”***

```rust
// immutable references
// example 0 (OK)
let s1: i32 = 10;
let s2: &i32 = &s1;

// example 1 (Error)
let s1: i32 = 10;
let s2: &i32 = &s1;
*s2 += 1;		// error: reference`s2` is a `&` reference
				// s2 是一个不可变引用，不能对其进行修改

// example 2 (Error)
let s1: i32 = 10;
let s2: &mut i32 = &mut s1;		// error: cannot borrow `s1` as mutable								
								// 不能创建一个 不可变变量 的 可变引用，因为 s1 是不可变的
*s2 += 1;	// error

// mutable references
let mut s3 = 20;
let s4 = &mut s3;
*s4 += 1;	// ok, *s4 == s3 == 21
```

*只有当 **原变量** 和 **借用来的引用** 都是 **可变类型(mutable)** 才可以使用这个 **可变引用** 修改原变量的值。*



## References Lifetimes

Rust compiler 负责自动管理引用的全部生命周期，也可以使用 `std::mem::drop()` 主动释放。

```rust
fn lifetime() {
    let mut x: i32 = 10;           // ┐ x 的生命周期开始
    let mut z: i32 = 20;           // │ z 的生命周期开始
    println!("original x: {}", x); // │
    {                              // │
        let mut y: &mut i32 = &mut x;  // ┐ y 的生命周期开始
                                       // │ y 借用 x
        *y += 1;                       // │
        println!("y: {}", y);          // │
        println!("x: {}", x);          // │ ← y 对 x 的借用在此结束
                                       // │   （最后一次使用）
        println!("original z: {}", z); // │
        y = &mut z;                    // │ y 现在借用 z
        *y += 1;                       // │
        println!("y: {}", y);          // │ ← y 对 z 的借用最后使用
    }                                  // ┘ y 的生命周期结束
    println!("z: {}", z);          // │ ✓ z 可以使用（y 已结束）
}                                  // ┘ x 和 z 的生命周期结束
```



### Three expressions

1. **词法作用域（Lexical Scope）**：根据作用域判断 引用 的生命周期。

2. **Non-Lexical Lifetimes (NLL)**：编译器自动判断 引用 的**实际使用范围**，引用 不再使用时自动结束其生命周期。

3. **显式生命周期标注** - 函数签名：

    ```rust
    // 编译器无法确定返回值的生命周期
    fn longest(x: &str, y: &str) -> &str {
        if x.len() > y.len() { x } else { y }
    }
    // error：缺少生命周期参数
    
    // ok：显式标注
    fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
        if x.len() > y.len() { x } else { y }
    }
    ```

    **`'a` 的含义**：返回值的生命周期至少和 `x` 和 `y` 中较短的那个一样长



### Detailed Explanation

1. 每个引用都有生命周期

    ```rust
    fn example() {
        let x = 5;          // x 的生命周期：'x
        let r = &x;         // r 的生命周期：'r
                            // r 引用的数据生命周期：'x
        // 约束：'r ≤ 'x （r 的生命周期不能超过 x）
    }
    ```

2. 引用不能超过被引用数据的生命周期

    ```rust
    fn dangling_reference() {
        let r;              // r 的生命周期：'r
        {
            let x = 5;      // x 的生命周期：'x
            r = &x;         // error：'r > 'x
        }                   // x 在这里结束
        println!("{}", r);  // r 还在用，但 x 已销毁
    }
    ```

3. 可变引用的独占性

    ```rust
    fn exclusive_borrow() {
        let mut x = 10;
        
        let r1 = &mut x;    // r1 独占 x
        // let r2 = &x;     // error：r1 还在使用
        *r1 += 1;
        println!("{}", r1); // r1 最后使用
        
        let r2 = &x;        // ✓ r1 已结束，可以创建 r2
    }
    ```

    

