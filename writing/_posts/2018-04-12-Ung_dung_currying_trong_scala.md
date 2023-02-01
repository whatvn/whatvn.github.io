---
title: Ứng dụng currying trong scala
date: 2018-04-01 12:00:00
---

# Currying

Theo như [wikimedia](https://en.wikipedia.org/wiki/Currying)

> In mathematics and computer science, currying is the technique of translating the evaluation of a function that takes multiple arguments (or a tuple of arguments) into evaluating a sequence of functions, each with a single argument. Currying is related to, but not the same as, partial application.


Như vậy, nếu như chúng ta có một function:

```scala
f(x, y) = x+y
```

thì ta có thể thực thi hàm tuần tự: `f(x, y) = h(x) + g(y)` trong đó `h(x) = x` và `g(y) = y` 
Nếu trong một thời điểm nào đó, ta có được giá trị `x=2`, thì khi đó `f(x,y) = x + y` trở thành `f(y) = 2 + y` hay `f(y) = 2 + g(y)` 

Trong scala, để biểu diễn f(x, y) thành một curried function, ta khai báo hàm thành `f(x)(y)` thay vì `f(x, y)`

```scala
def f(x: Int)(y: Int): Int = x + y 
```

Khi có được giá trị của `x`, ta gọi `f` như sau:

```scala
scala> def f(x:Int)(y: Int): Int = x + y
f: (x: Int)(y: Int)Int

scala> val x = 2
x: Int = 2

scala> val gy = f(x)
gy: Int => Int = <function1>

scala> gy(2)
res0: Int = 4
```

Để ý return type khi gọi `f(x)_`, ta có được `function1`, function1 là một `type` được định nghĩa sẵn của scala [Function1](http://www.scala-lang.org/api/2.12.0/scala/Function1.html), hiểu đơn giản `Function1` là function có 1 tham số đầu vào, với `apply` method là 

```scala
def apply(y: Int) = 2 + y
```

Mở rộng cách triển khai bên dưới của curried function `f(x)(y)` bên trên, ta có thể hiểu khi gọi `f(x)` với `x=2`, `f(x)` sẽ được biểu diễn như sau:

```scala
def f(x: Int): Function1[Int, Int] = {
      new Function[Int, Int] {
        def apply(y: Int): Int = {
          y + x
        }
      }
}
```

Khi gọi `f` với `x=2`, ta có kết quả tương tự như trên: 

```scala
scala> val gy = f(2)
gy: Int => Int = <function1>

scala> gy(1)
res18: Int = 3
```

# Ứng dụng 

# 1. Cách đơn giản nhất

Khi lập trình phần mềm, chúng ta thường gặp trường hợp phải gọi một hàm với hai hoặc nhiều tham số đầu vào, nhưng tại thời điểm nhất định, chỉ có một giá trị duy nhất. Ví dụ như hàm `f(x, y)` bên trên. Tại thời điểm T1 ta chỉ có giá trị của x, khi gọi `f(x = 2)` với ngôn ngữ như `Java` hoặc `Python` sẽ gặp lỗi, vì các ngôn ngữ này không hỗ trợ curried function. Ta buộc phải truyền giá trị của x đi khắp nơi hoặc lưu lại ở một biến global nào đó, sau đó ở thời điểm `T2` tìm được giá trị của `y` mới gọi `f` để xử lí. 


Thử tưởng tượng nếu như chúng ta có một flow xử lí thông tin như sau: 

```
1. Request đến một api endpoint để lấy thông tin tất cả người dùng 
2. Parse response trả về để lấy id người dùng
3. Tạo request tiếp theo đến api endpoint khác để lấy thông tin cụ thể của từng người dùng (UserDetail)
4. Parse UserDetail để lấy địa chỉ email của người dùng
5. Gọi đến api endpoint cuối cùng với userid để lấy nội dung email, gửi nội dung email cho người dùng với id tương ứng.
```

Cho đến cuối cùng flow trên sẽ được đơn giản hoá thành một hàm như bên dưới: 

```scala
def process(url: String)(op: HttpResponse => Unit) = ???
```

Hàm này nhận vào một Http request, và xử lí Http Response, và trả về giá trị Unit (thay vì void type ở C/C++/Java). Tham số `op: HttpResponse => Unit` là bất cứ hàm nào nhận vào tham số là `HttpResponse` và trả về `type` là `Unit`.
Nếu như dùng currying, cách triển khai như sau: 

```scala
val totalUserApi = "http://apiserver.com/totalusers"
val userDetailApi = "http://apiserver.com/getuserdetail"
val emailApi = "http://apiserver.com/getemail"


def process(url: String)(op: HttpResponse => Unit) = {
  val request = HttpRequest(url)
  val response = MakeRequest(request) getResponseEntity 
  op(response)
}

def sendEmail(email: String)(httpResponse: HttpResponse) = {
  emailContent = getContent(httpResponse)
  send(email, emailContent)
}

def getEmail(httpResponse: HttpResponse) = {
   val email = getEMail(httpResponse)
   val sender = sendEMail(email) _ // currying
   process(s"$emailApi?)(sender)
}

def parseToTalUsersResponse(httpResponse: HttpResponse) = {
  val ids = do_something_to_parse_json_string_in_response_entity(httpResponse)
  ids.foreach({ id => 
    process(s"$userDetailApi?id=$id")(getEmail) 
  }) 

}

// do 1st step: 
process(totalUserApi)(parseToTalUsersResponse)

```

Tất cả các bước trên đều là async, nhưng vẫn *readable*. Nếu không sử dụng currying, chúng ta buộc phải tìm cách nào đó để chuyển flow này thành tuần tự, hoặc phải viết thêm một hàm trả về `function` : `HttpResponse => Unit` như bên dưới:

```scala
def creatSender(email: String): HttpResponse => Unit = {
  response => send(email, response)
}
```

Sau đó viết lại hàm getEmail: 

```scala
getEmail(httpResponse: HttpResponse) = {
   val email = getEMail(httpResponse)
   val sender = creatSender(email)  // no currying
   process(s"$emailApi?)(sender)
}
``` 

Phần lớn mọi người khi nhìn vào hàm có return type là `function` luôn cảm thấy khó hiểu. 


# 2. implicit parameter

Internal implementation của scala và các thư viện viết bằng scala ứng dụng currying rất nhiều, chủ yếu thông qua `implicit parameter`.
**Implicit parameter** trong scala là một ứng dụng của currying + implicit cho phép compiler insert một tham số còn thiếu vào lời gọi hàm.

Có thể hiểu một cách tổng quát, với Scala, một hàm được khai báo với 2 hoặc nhiều tham số đầu vào, nhưng ta có thể sử dụng hàm này với chỉ 1 tham số đầu vào duy nhất, phần còn lại sẽ được compiler lo liệu, bằng cách tìm kiếm implicit trong các giá trị implicit phù hợp được define ở phạm vi cho phép, và dùng implicit này để insert vào lời gọi hàm. 

`Implicit parameter` đuợc đánh giá là ứng dụng advance của scala, tuy nhiên do chúng ta sẽ gặp rất nhiều khi làm việc với scala cũng như các thư viện viết bằng scala, nên cũng nên tìm hiểu về khái niệm này. 

Một ví dụ ứng dụng `implicit parameter`: khi làm việc với một thư viện hoặc hàm cần tham số đầu vào là một type tương ứng 

```scala
case class Custom(words: String) 

```
và một method `say` yêu cầu `Custom` là tham số đầu vào, chúng ta dùng `say` như sau:

```scala
val words = "Hello scala"
val obj = Custom(words)
say(obj)
```

Dễ dàng nhận thấy `words` được wrap bên trong Custom chỉ để người thiết kế thư viện ấn định một **type**, việc này cần thiết đối với trường hợp ứng dụng [Pattern matching](http://tinytechie.net/#/_posts/Pattern_matching_and_partial_funtion), tuy nhiên đối với người sử dụng, việc tính toán để có được kết quả là `String`, sau đó tất cả mọi lần sử dụng đều phải `new Custom object` với `String` có được là khá phiền phức, nếu như ta chỉ sử dụng một object type duy nhất là `Custom`, có thể sử dụng implicit parameter trong trường hợp này, và do implicit parameter áp dụng currying, đây cũng được coi là một ứng dụng của currying:

Define một method để implicit convert `String` to `Custom`

```scala 
implicit def stringToCustom(s: String) = Custom(s)
// và implement hàm cần sử dụng say method với implicit parameter

def sayToTheWorld(words: String)(implicit worldToObject: String => Custom) = {
  say(words)
}
```

Dùng method đã define 

```scala
val words = "Hello Scala"
sayToTheWorld(words)
```

Để ý ta sử dụng hàm `say` với tham số đầu vào là `words`, với type là `String`. Khi compiler biên dịch đoạn code trên, gặp `sayToTheWorld(words)` sẽ bị lỗi, trước khi dừng việc biên dịch và thông báo lỗi, compiler sẽ tìm kiếm trong scope đang thực thi xem có sự hiện diện của một implicit nào có input là `String` và output là `Custom` không? Nếu có, implicit này sẽ được insert tự động vào lời gọi hàm `sayToTheWorld` và ngay cả `say` trong body của hàm `sayToTheWorld`

Bản thân Core implementation của scala cũng sử dụng vô số implicit parameter, từ việc [chuyển đổi](http://docs.scala-lang.org/overviews/collections/strings) `Java String` sang `IndexSeq`, cho đến [Fuctures/Promises](http://docs.scala-lang.org/overviews/core/futures.html) với `ExecutionContext`

# Kết 

Bài này chỉ nhắc đến 2 ứng dụng nhỏ của currying, và sẽ được cập nhật khi có thời gian. Currying là một trong những feature rất hay khi chuyển từ Java sang scala, ngoài rất nhiều những khái niệm khác như: Object, trait, case class, pattern matching...


 
 


 
