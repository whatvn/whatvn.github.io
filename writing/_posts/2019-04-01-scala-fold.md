---
title: Tổng quan fold, foldLeft & foldRight
date: 2019-04-01 12:00:00
---

![foldLeft vs foldRight (hình trộm từ Internet)](https://1.bp.blogspot.com/-Lj26_YcdWwU/V6Fz99p-AMI/AAAAAAAAIuc/zAGMUmfF0JEsgPvVPWvFUMmcm8big4hKgCLcB/s640/foldlfoldr1.jpg)  

***foldLeft*** được implement như sau:

```
override /*TraversableLike*/ def foldLeft[B](z: B)(@deprecatedName('f) op: (B, A) => B): B = {
  var acc = z
  var these = this  while (!these.isEmpty) {
    acc = op(acc, these.head)
    these = these.tail
  }
  acc
}
```

Như vậy foldLeft là một curried function với tham số đầu vào (giá trị bắt đầu) là z có kiểu là B, và một function op. op có tham số đầu vào là 2 biến có kiểu B và A,  và giá trị trả về có kiểu là B. 
Hàm foldLeft dùng while loop (không dùng đệ quy), và thực hiện op trên biến mutable acc và phần tử đầu tiên của list these (these.head), đồng thời gán lại giá trị cho list these cho đến khi length của these = 0 (empty).

Nhìn vào hình minh hoạ đầu bài. Để minh hoạ quá trình chạy của foldLeft, ta thực hành trên một ví dụ như sau: 


```
List("1","2","3").foldLeft("")((x, y) => { 
  print("x="+x); print(" & "); 
  print("y="+y); 
  println(" => x+y=" + x+y); 
  x + y
})
```

Khi chạy, đoạn code trên cho output:

```
x= & y=1 => x+y=1
x=1 & y=2 => x+y=12
x=12 & y=3 => x+y=123
res44: String = 123
```

***Kết quả trả về có kiểu String***,   type B trong khai báo foldLeft. Method "+" chúng ta dùng ở đây là string concatenation, không phải method addition trong toán học.

Vậy hàm trên bắt đầu bắt giá trị khởi đầu start (z) là: `"" ` được gán cho x, và y là `List("1", "2", "3").head => "1"`, thực hiện `op("", "1") => "" + "1" `là cộng chuỗi ta có kết quả 1, dùng kết quả này chạy tiếp vối op, ta có sơ đồ sau:

```
op("", "1") => op("1", "2") => op("12", "3")
"1"         => "12"         => "123"  
```

***fold***: Bài này nhắc đến foldLeft trước fold vì fold là một dạng triển khai của foldLeft(1)

```
def fold[A1 >: A](z: A1)(op: (A1, A1) => A1): A1 = foldLeft(z)(op)
```

Chỉ khác là fold thao tác trên kiểu A1, là một supertype của A, và trả về kiểu A1. Sự khác nhau ở đây là gì? Nếu ta dùng fold với op có tham số đầu vào là `Int`, thì type A1 sẽ là `AnyVal`, do `Int` là một implement của `AnyVal`. Còn nếu dùng fold với op có tham số đầu vào là String, thì type A1 sẽ là `Any`, do String của scala kế thừa lại của Java, và trong Scala tất cả các kiểu đều kế thừa từ `Any`.

Vậy nếu ta fold trên một List có kiểu `String`, và dùng op với thông số đầu vào là kiểu `Int`, thì A1 sẽ có type là `Any`, do `String` và `Int` có chung supertype là `Any`. Và do `Any` không có method `+` (cả concatenation và addition), nên ta không thể dùng fold như bên dưới:

```
List("1","2","3").fold(0)((x, y) => {
  print("x="+x); print(" & ");
  print("y="+y);
  println(" => x+y=" + x+y);
  x + y
})
```

Hoặc


```
List("1","2","3").fold(0)((x, y) => {
  print("x="+x); print(" & ");
  print("y="+y);
  println(" => x+y=" + x+y);
  x + y.toInt
})
```

Ta chỉ có thể dùng fold với biến đầu vào cho op là String, hoặc Unit hoặc char

```
List("1","2","3").fold("")((x, y) => {
  print("x="+x); print(" & ");
  print("y="+y);
  println(" => x+y=" + x+y);
  x + y
})
```

Trong trường hợp muốn dùng fold trên một `List[String] `và kết quả trả về là `Int,` ta buộc phải dùng `foldLeft` hoặc `foldRight`.

***foldRight***: được implement như sau

```
override def foldRight[B](z: B)(op: (A, B) => B): B =
  reverse.foldLeft(z)((right, left) => op(left, right))
``` 

Tham số đầu vào của foldRight hoàn toàn giống với foldLeft. Cách implement có khác một chút:

`reverse` là một method được implement bên trong class List.scala, sẽ đảo ngược List đang làm việc. Nhìn vào có vẻ khó hiểu nếu như ta nhìn theo logic thông thường ta dễ nhầm lẫn foldLeft được gọi như một method của reverse, nhưng thật ra ta đang dùng foldLeft như method của List được trả về khi gọi hàm reverse. Tương tự như ta dùng map như bên dưới: 

```
List(1, 2, 3).reverse.map(_ + 1)
```

Tương tự như ví dụ ban đầu, áp dụng foldRight:

```
List("1","2","3").foldRight("")((x, y) => {
	print("x="+x); print(" & ");
	print("y="+y);
	println(" => x+y=" + x+y);
	x + y
})
```

Khi chạy đoạn code trên, ta sẽ có output:

```
x=3 & y= => x+y=3
x=2 & y=3 => x+y=23
x=1 & y=23 => x+y=123
res4: String = 123
```

Kết quả trả về cùng kiểu `String `như foldLeft, như thay vì op (x + y) được apply ở foldLeft là "" + List("1", "2", "3).head thì được thay bằng List("1", "2", "3").reverse.head + "", ta có sơ đồ sau:

```
op("", "3") => op("3", "2") => op("23", "1")
```

và vì op sẽ đảo thứ tự của 2 biến đầu vào nên ta sẽ có

```
"3" + ""     =>  "2" + "3"     => "1" + "23" = "123"
```

(1)Scala có 2 dạng triển khai của method fold, một dành cho các cấu trúc dữ liệu dạng chuỗi thông thường, và một phức tạp hơn dành cho parallel collection, sẽ được nói đến trong một bài khác. 
