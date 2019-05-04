---
title: Scala basic - tổng quan về hàm trong scala
date: 2018-02-01 03:00:00
---

Scala cho chúng ta nhiều cách để khai báo function, trong đó có:

***Local function*** (ví dụ bên dưới có vẻ ngu ngốc, vì ta hoàn toàn có thể dùng filter với anonymous function `x.filter(_ > 1)` , nhưng nó nói lên được ý nghĩa của local function và phần nào làm rõ nghĩa được filter method của List

```
def filterInt(x: List[Int]): List[Int] = {
   def f(y: Int, z: Int) = {
     y > z
  }
  x.filter(f(_, 1))
}
```

***First-class Function***: Không như cách chúng ta khai báo một hàm và gọi hàm đó bằng tên, như ví dụ bên trên: ```filter(List(1, 2, 3)```, Scala còn cho phép khai báo hàm không tên (unamed literal function) và truyền vào các hàm khác như một giá trị (function value). Một cách để biểu diễn loại hàm này là viết lại hàm bên trên sử dụng filter method của List như sau:

```
a.filter((x) => x > 1)
```

hoặc lược 2 dấu  `( `và `) ` : `a.filter(x => x > 1)`
như vậy
`(x) => x > 1` hoặc `x => x > 1` được dùng như một biến đầu vào, scala gọi là function value. Mở rộng cách dùng function value không dùng built-in method của scala:

```
def addMore(op: (Int) => Int, y: Int ) = {
  op(y)
}

val ten = (x: Int) => x + 10
val two = (x: Int) => x + 2

addMore(ten, 1) // 11
addMore(two, 1) // 3
```

***Partial applied function***: Trong một vài trường hợp khi khai báo một hàm nhưng ta chưa đủ các dữ liệu mà chỉ có được một phần trong danh sách dữ liệu, có thể dùng partial applied function - hàm với các biến cục bộ:

```
def sum(x: Int, y: Int): Int = {
  x + y
}
```

Có thể dùng sum như sau trong trường hợp chỉ có giá trị của x:

```
val plusOne = sum(1, _:Int)
```

sau các bước tính toán và có giá trị của y, lấy kết quả:

```
plusOne(10) // 11
```

***Closure***: Thông thường khi khai báo các hàm, chúng ta thường sử dụng các biến được khai báo bên trong ngữ cảnh của hàm đó, như x và y trong ví dụ bên trên. Các biến này được gọi là `bound variable` hay là các biến đóng. Ngoài khái niệm biến đóng, Scala (hay các ngôn ngữ khác cho ta một khái niệm khác là biến tự do, ví dụ:

```
var i = 5;
val f = (x: Int) => x + i
f(10) //15
```

Khái niệm closure (closing + capturing free variables)  được tạo nên bởi thuật ngữ chỉ một function value được tạo ra khi chương trình chạy (runtime). Với ví dụ trên, khi i thay đổi, giá trị của f cũng sẽ thay đổi theo (chú ý ta không khai báo i là val mà là var) 
Nếu thay đổi giá trị của i

```
i = 6
println(f(10))
```

Thì giá trị được in ra là `6`, chứ không phải là `15`
Ta áp dụng closure như sau: 

```
def increase(more: Int) = {
  (x: Int) => x + more
}
val f1 = increase(10)
println(f1(3)) //13
val f2 = increase(11)
println(f2(3)) //14
```

***Các cách gọi hàm khác***: 
Scala cho phép tham số cuối của một hàm có thể được lặp, chú ý ta chỉ có thể lặp lại tham số cuối, cho nên ta có thể khai báo:

```
def repeatSecondArgs(first: Int, args: Any*): Unit = {
  println(first + args.mkString(" ", " ", ""))
}
repeatSecondArgs(1000, "Hello", "World") //prints "1000 Hello World"
```

Nhưng không thể khai báo:

```
// Compiler sẽ báo lỗi 
def printFirstArgs(first: Int*, second: String): Unit = {
  println(args.mkString(" "))
} 
```

Ngoài ra chúng ta có thể truyền tham số với tên biến đầu vào:

```
class Student(val Name: String, val age: Int) {
  require(age >= 18)
  override def toString(): String = {
    "[" + Name + ":" + age.toString + "]"  }
}
def createStudent(name: String, age: Int): Student = {
  new Student(name, age)
}
val hung = createStudent("Hung", 30)
val hung1 = createStudent("Hung", age = 30)
val hung2 = createStudent(age = 30, name = "Hung")

println(hung) //[Hung:30]
println(hung1) //[Hung:30]
println(hung2) //[Hung:30]
```

Hoặc khai báo hàm với tham số mặc định:

```
def getSalary(workingHours: Int, factor: Float = 1.0F, payPerHour: Float = 12.0F): Float = {
  payPerHour*workingHours*factor
}
println(getSalary(60)) // 72.0
println(getSalary(60, factor = 1.2F)) // 864.00006
println(getSalary(60, payPerHour = 12.5F)) // 750.0
```

Partial Function: xem bài trước

***Currying***: là một cách biến đổi phương thức thực thi của một hàm có đầu vào là nhiều tham số thành một chuỗi các hàm riêng lẻ với đầu vào là một tham số duy nhất. Ví du với hàm bên dưới

```
def sum(x: Int, y: Int): Int = {
  x + y
}
```

Để chuyển sum thành curried function trong scala ta khai báo hàm lại như sau:

```
def sum(x: Int)(y: Int) = {
  x + y
}
```

Dùng sum được khai báo bên trên:

```
val plus10 = sum(10)_
println(plus10(20))
```


Thậm chí có thể khai báo nhiều hơn 2 danh sách tham số:

```
def sum(x: Int)(y: Int)(z: Int) = {
  x + y + z
} 
```

Để hiểu được nhu cầu sử dụng curried function trong thực tế phải có một bài riêng để nói về currying. Khi nhìn vào cách sử dụng currying thì ta thấy curry tương tự như function value. Như ví dụ bên dưới, khai báo hàm sort một List các phần tử sử dụng merge sort. Với currying ta dùng như sau:

```
def msort[T](cmp: (T, T) => Boolean) (xs: List[T]):List[T] = {
  def merge(xs: List[T], ys: List[T]): List[T] =
    (xs, ys) match {
      case (xs, Nil) => xs
      case (Nil, ys) => ys
      case (x::xs1, y::ys1) =>  {
        if (cmp(x, y)) x:: merge(xs1, ys)
        else y::merge(xs, ys1)
      }
  }
  val n = xs.length / 2  if (n == 0) xs
  else {
    val (ys, zs) = xs splitAt n
    merge(msort(cmp)(ys), msort(cmp)(zs))
  }
}
```

Khi dùng function value cũng tương tự:

```
def nonCurriedMsort[T](cmp: (T, T) => Boolean, xs: List[T]):List[T] = {
  def merge(xs: List[T], ys: List[T]): List[T] =
    (xs, ys) match {
      case (xs, Nil) => xs
      case (Nil, ys) => ys
      case (x::xs1, y::ys1) =>  {
        if (cmp(x, y)) x:: merge(xs1, ys)
        else y::merge(xs, ys1)
      }
    }
  val n = xs.length / 2  if (n == 0) xs
  else {
    val (ys, zs) = xs splitAt n
    merge(nonCurriedMsort(cmp, ys), nonCurriedMsort(cmp, zs))
  }
}
val l = List(1, 3, 2, 5, 4, 9, 6)
def cmp(x: Int, y: Int) = x < y
msort(cmp) (l)
nonCurriedMsort(cmp, l)
val a = msort(cmp)_ 
val b = nonCurriedMsort(cmp, _: List[Int])
a(l) 
b(l)
```

Ví dụ về merge sort lấy từ Programming scala 2nd. Hi vọng sẽ có cơ hội nói về một real world case sử dụng curried function trong vài ngày tới.
