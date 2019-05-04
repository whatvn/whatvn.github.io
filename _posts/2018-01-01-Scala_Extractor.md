---
title: Scala Basic - Extractor 
date: 2018-01-03 03:00:00
---

```
package extractor

object RegexExtractor {
  def main(args: Array[String]): Unit = {
    val video = ".*\\/(.*)-([0-9]+).*".r
    def process(path: String) = path match {
      case video(name, id) => println(name, id)
      case _ => ()
    }
    val videoUrl = "/phim/night-light-i2-4538/"
    process(videoUrl)
  }
}

```

# Representation independence

Chúng ta đã nói về [Scala pattern matching](http://tinytechie.net/#/_posts/Pattern_matching_and_partial_funtion), và phần nào thấy được sự tiện dụng cũng như việc dùng `case class` kết hợp với `sealed class` giúp người lập trình làm việc nhanh hơn, hạn chế số dòng code và các lỗi tiềm ẩn có thể xảy ra với những logic phức tạp. 

Mặc dù **case class** vô cùng hữu dụng, tuy nhiên sử dụng `case class` đồng nghĩa với việc chúng ta phơi bày internal implementation của dữ liệu cho client (người sử dụng thư viện), mà đôi khi việc này là không cần thiết và dẫn tới hệ luỵ: một khi chúng ta muốn thay đổi cách xây dựng dữ liệu, mã nguồn người dùng cũng phải thay đổi theo.

Chú ý việc sử dụng **case class** tương ứng với việc chúng ta define class với public field, với scala thì các public field này là immutable. Cách biểu diễn dữ liệu (data representation) và triển khai dữ liệu (data implementation) không độc lập. Xem qua ví dụ bên dưới:

```
abstract class BaseVideo
object Video  {
	private class VideoImpl(val id: long, val metadata: VideoMetadata) extends BaseVideo
	def apply(id: long, metadata: VideoMetadata):BaseVideo = new VideoImpl(id, metadata) 
}

```

Khi client sử dụng thư viện với type `BaseVideo` qua `apply` method của Singleton object `Video`: 

```
val video = Video(142424434l, someMetadata)
```

Khi cần thay đổi cách xây dựng `Video` type, chúng ta chỉ cần thay đổi mã nguồn của thư viện, và client không cần biết đến sự thay đổi này, và cũng không cần phải sửa đổi mã nguồn để có thể làm việc với thư viện mới

```
abstract class BaseVideo

object Video  {
	private class VideoImpl2(val id: long, val metadata: VideoMetadata) extends BaseVideo
	def apply(id: long, metadata: VideoMetadata): BaseVideo = {
		val id2 = id + 100   // add 100 to id
		new VideoImpl2(id2, metadata)  // use new constructor 
	}
}
```

# Extractor 

**case class** chỉ có thể làm việc với constructed object, có nghĩa là chúng ta chỉ có thể áp dụng pattern matching dối với những object đã được định nghĩa với type cụ thể nào đó: 

```
abstract class X

case class Y(y: String) extends X
case class Z(Z: String) extends X

def printX(x: X) = x match {
	case Y(y) => println(y)
	case Z(z) => println(z)
}

```

Tuy nhiên trong nhiều tình huống chúng ta muốn xử lí trực tiếp trên dữ liệu nhận được như đoạn mã nguồn đầu bài, nhận HttpRequest, và dùng pattern matching với url được request, và bóc tách ra id + name của video, loại bỏ các dữ liệu không cần thiết, như phần đầu của url `/phim/`, dấu `-` và `/` ở cuối cùng, thì case class không áp dụng được. Cách được dùng gọi là **extractor** trong scala. 

Để ý chúng ta khai báo một `regular expression` là `video`, nhưng cách sử dụng lại như dùng `constructor pattern` của `case class`, làm được như vậy bởi vì `Regex type` của scala có implement sẵn các method `unapply` và `unapplySeq`. 
Hiểu một cách đơn giản, các method trên là **đảo ngược của constructor**, nếu như constructor (apply method) là nhận vào một seq/tuple/field để xây dựng nên một String, thì `unapply` sẽ làm ngược lại, nhận vào **String** và bóc tách các field cần thiết. 

## unapply method

```
object URL {
  def unapply(url: String): Option[(String, String)] = {
    val parts = url.split("_")
    if (parts.length == 2) Some(parts(1), parts(0)) else None
  }
}

def extract(url: String) = url match {
      case URL(id, name) => println("ohooo")
      case _ => println("nothing")
    }
    val url = "here-is-a-title_23423424"
    extract(url) // prints "23423424"
    val url1 = "someothertitle-2432568568"
    extract(url1) //prints "nothing"
```
Khi một object được khai báo với `unapply` method, object đó trở thành extractor. `unapply` method phải được triển khai theo nguyên tắc sau: 

- return **type** của unapply method phải là: **Option** hoặc **Boolean**
- unapply method phải cover trường hợp input thoả/hoặc không thoả điều kiện, nếu thoả return **Some(tuple)** (1) hoặc `true`, ngược lại return **None** hoặc **false** 
- unapply method luôn luôn nhận biến đầu vào 

> (1) scala không có tuple với một phần tử, cho nên trong trường hợp **unapply** chỉ trả về một giá trị duy nhất, chúng ta thay đổi return type của method thành **Option[SomeType]** và wrap giá trị trả về bằng **Some(thing)**

Trường hợp **unapply** trả về **Boolean**, scala cho phép gọi extractor với `zero parameter` sử dụng [variable binding](http://www.scala-lang.org/files/archive/spec/2.11/08-pattern-matching.html#variable-patterns), syntax này không trong sáng cho lắm, khó hình dung và rối rắm:

```
object NewVideo {
	def unapply(id: String): Boolean = {
		id.toInt > 200000
	}
}
``` 

Ví dụ trên chúng ta định nghĩa một object `NewVideo` với `unapply` method để kiểm tra xem video có lơn hơn *200000* hay không, để ý return type của method này là **Boolean**, kết hợp với ví dụ trước đó, ta sử dụng `NewVideo` extractor như sau: 

```
 def extract(url: String) = url match {
      case URL(id @ NewVideo() , name) => println(id)
      case _ => println("nothing")
}
val url = "here-is-a-title_23423424"
extract(url)  //prints "23423424"
val url1 = "some-other-title_24325"
extract(url1) // prints "nothing" 
```   
Theo quan điểm cá nhân, chúng ta chỉ nên sử dụng pattern này khi thiết kế thư viện, còn ngoài ra khi làm việc nhóm thì sẽ nhiều trường hợp người khác nhìn vào mã nguồn mà không hiểu đoạn mã nguồn này đang làm việc gì.

## unapplySeq method

**unapply** method hữu dụng trong trường hợp chúng ta muốn dùng pattern matching với số lượng phần tử nhất định, hoặc muốn kiểm tra xem biến đầu vào có thoả một điều kiện cụ thể nào đó. 

Tuy nhiên khi làm việc với biến đầu vào đòi hỏi phải so sánh nhiều trường hợp hơn thì **unapply** không đáp ứng được. Do vậy, ta dùng **unapplySeq**. 
Ví dụ: báo vnexpress.net biểu diễn các url theo dạng:

```
http://vnexpress.net/photo/giao-thong/buy-t-nhanh-bi-bua-vay-tren-duong-pho_3522623.html
http://vnexpress.net/tin-tuc/thoi-su/cuoc-tran-tro-dua-da-nang-len-trung-uong-20-nam-truoc_3520938.html
http://vnexpress.net/tin-tuc/quoc-te/dong-tien-trung-quoc-co-the-bien-dong-manh-nam-nay_3522935.html
```
Để kiểm tra một tin thuộc thể loại nào, nằm trong chuyên mục con nào... với các dạng url có vẻ đa dạng, **unapply** không thể áp dụng. 

```
object News {
  def unapplySeq(fullURL: String): Option[Seq[String]] = {
     Some(fullURL.split("/"))
  }
}

def extract(url: String) = url match {
  case News("tin-tuc" , "thoi-su", i) => println(s"$i thuoc tin tuc, phan muc quoc te")
  case News("photo", "giao-thong", i) => println(s"$i thuoc photo, phan muc giao thong")
  case _ => println("the loai khac")
}

val url = "photo/giao-thong/buy-t-nhanh-bi-bua-vay-tren-duong-pho-3522623.html"
val url1 = "tin-tuc/thoi-su/cuoc-tran-tro-dua-da-nang-len-trung-uong-20-nam-truoc_3520938.html"
extract(url)  // prints "buy-t-nhanh-bi-bua-vay-tren-duong-pho_3522623.html thuoc photo, phan muc giao thong"
extract(url1) // prints "cuoc-tran-tro-dua-da-nang-len-trung-uong-20-nam-truoc_3520938.html thuoc tin tuc, phan muc quoc te"
```

Như vậy nguyên tắc của **unapplySeq** như sau:

- unapplySeq luôn nhận biến đầu vào
- return type của unapplySeq là **Option[Seq[SomeType]]** hoặc 
- return type của unapplySeq có thể là **tuple** với số lượng phần tử cố định, như sau: 

```
object News {
  def unapplySeq(fullURL: String): Option[(String, String, Seq[String])] = {
    val parts = fullURL.replace(".html", "").split("/")
    if (parts.length == 3) Some(parts(0), parts(1), parts(2).split("_")) else None
  }
}

def extract(url: String) = url match {
  case News("tin-tuc" , "thoi-su", i @ _* ) => println(s"${i.last} thuoc tin tuc, phan muc quoc te")
  case News("photo", "giao-thong", i @ _*) => println(s"${i.last} thuoc photo, phan muc giao thong")
  case _ => println("the loai khac")
}

val url = "photo/giao-thong/buy-t-nhanh-bi-bua-vay-tren-duong-pho_3522623.html"
val url1 = "tin-tuc/thoi-su/cuoc-tran-tro-dua-da-nang-len-trung-uong-20-nam-truoc_3520938.html"
extract(url)  // 3522623 thuoc photo, phan muc giao thong
extract(url1) // 3520938 thuoc tin tuc, phan muc quoc te

```

## Sử dụng cùng lúc unapply và unapplySeq

Cũng ví dụ bên trên, ta có thể áp dụng cùng lúc **unapplySeq** và **unapply**: 

```
object NewsUrl {
  def unapplySeq(fullURL: String): Option[Seq[String]] = {
    Some(fullURL.replace(".html", "").split("/"))
  }
}

object NewsDetail {
  def unapply(arg: String): Option[(String, String)] = {
    val parts = arg.replace(".html", "").split("_")
    if (parts.length == 2) Some(parts(0), parts(1)) else None
  }
}

def extract(url: String) = url match {
  case NewsUrl("tin-tuc" , "thoi-su", NewsDetail(name, id) ) => println(s"$id thuoc tin tuc, phan muc quoc te")
  case NewsUrl("photo", "giao-thong", NewsDetail(name, id)) => println(s"$id thuoc photo, phan muc giao thong")
  case _ => println("the loai khac")
}

val url = "photo/giao-thong/buy-t-nhanh-bi-bua-vay-tren-duong-pho_3522623.html"
val url1 = "tin-tuc/thoi-su/cuoc-tran-tro-dua-da-nang-len-trung-uong-20-nam-truoc_3520938.html"
extract(url) // 3522623 thuoc photo, phan muc giao thong
extract(url1) // 3520938 thuoc tin tuc, phan muc quoc te
```

# Kết 

Extractor cho phép chúng ta có thể sinh ra các pattern tuỳ biến khi lập trình với scala mà không nhất thiết phải sử dụng `case class`, và không cần phả sử dụng case class, ta có thể tách biệt giữa "data representation" và "data implementation", giúp việc thiết kế thư viện tuỳ biến hơn, client không cần phải thay đổi mã nguồn nếu như người phát triển thư viện thay đổi các implementation của dữ liệu. 

Tuy nhiên trong nhiều trường hợp, sử dụng **case class** sẽ an toàn hơn, giúp tránh được lỗi khi sử dụng `pattern maching`, giúp cho chúng ta lập trình nhanh hơn, mã nguồn ngắn hơn. Và mã nguồn ngắn hơn -> ít lỗi hơn. 
