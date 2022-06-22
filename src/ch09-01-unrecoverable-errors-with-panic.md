## Lỗi không thể khôi phục với `panic!`

Đôi khi, những điều tồi tệ xảy ra trong mã của bạn và bạn không thể làm gì với nó.
Trong những trường hợp này, Rust có macro `panic!`. Có hai cách để gây ra lỗi trong
thực tế: bằng cách thực hiện một hành động khiến mã của chúng ta bị lỗi (chẳng hạn như
truy cập vào một vị trí ngoài mảng) hoặc bằng cách gọi chỉ định rõ ràng macro `panic!`.
Trong cả hai trường hợp, chúng ta gây ra lỗi trong chương trình. Theo mặc định, những lỗi này sẽ
in một thông báo lỗi, giải phóng những thứ đã tạo (unwind), dọn dẹp ngăn xếp (stack)
và thoát chương trình. Thông qua một biến môi trường, bạn cũng có thể để Rust hiển thị
ngăn xếp của những lời gọi thực thi khi xảy ra lỗi để dễ dàng tìm ra nguồn gốc của lỗi này.

> ### Giải phóng (unwind) stack hoặc hủy bỏ (abort) chương trình khi lỗi xảy ra
>
> Theo mặc định, khi xảy ra lỗi, chương trình sẽ bắt đầu *unwinding*, có nghĩa là
> Rust sẽ đi ngược lại các thông tin ngăn xếp và dọn dẹp dữ liệu của mỗi chức năng
> mà nó gặp. Tuy nhiên, việc di chuyển và dọn dẹp này tốn rất nhiều công việc.
> Do đó, Rust cho phép bạn chọn giải pháp thay thế là ngay lập tức *aborting*,
> có nghĩa là kết thúc chương trình mà không cần dọn dẹp gì hết.
>
> Bộ nhớ mà chương trình đã sử dụng sẽ được dọn dẹp bởi hệ điều hành.
> Nếu trong dự án, bạn cần làm cho tập tin nhị phân kết quả càng nhỏ càng tốt,
> bạn có thể chuyển từ loại unwind sang abort khi có lỗi xảy ra,
> bằng cách thêm `panic = 'abort'` vào phần thích hợp `[profile]` trong tập tin *Cargo.toml*.
> Ví dụ: nếu muốn hủy bỏ chương trình (abort) với lỗi xảy ra, trong chế độ phát hành (release),
> thêm như sau:
>
> ```toml
> [profile.release]
> panic = 'abort'
> ```

Hãy thử gọi macro `panic!` trong chương trình đơn giản này:

<span class="filename">Filename: src/main.rs</span>

```rust,should_panic,panics
{{#rustdoc_include ../listings/ch09-error-handling/no-listing-01-panic/src/main.rs}}
```

Khi bạn chạy chương trình, sẽ thấy những thứ như:

```console
{{#include ../listings/ch09-error-handling/no-listing-01-panic/output.txt}}
```

Lời gọi `panic!` gây ra thông báo lỗi trong hai dòng cuối cùng.
Dòng đầu tiên hiển thị thông báo lỗi của chúng ta và vị trí trong mã nguồn nơi xảy
ra lỗi: *src/main.rs:2:5* cho biết đó là dòng thứ hai, ký tự thứ năm của tập tin *src/main.rs*.

Trong trường hợp này, dòng được chỉ ra là một phần trong mã của chúng ta và nếu
chúng ta đi đến dòng đó, sẽ thấy lệnh gọi macro `panic!`. Trong các trường hợp khác,
lệnh gọi `panic!` có thể nằm bên trong mã khác mà chương trình của chúng ta gọi tới,
và tên tệp với số dòng được thông báo lỗi sẽ là mã của người khác, đó là nơi mà macro `panic!`
được gọi, không phải là do dòng mã của chúng ta gọi `panic!`. Chúng ta có thể sử dụng dấu vết
của các hàm mà đã gọi `panic!`, để tìm ra phần mã nào của chúng ta đang gây ra sự cố.
Tiếp theo, chúng ta sẽ thảo luận chi tiết hơn về các dấu vết (backtrace).

### Sử dụng truy vết một `panic!`

Hãy xem xét một ví dụ khác để xem nó như thế nào khi `panic!` được gọi từ thư viện do lỗi
chương trình (bug) trong mã của chúng ta thay vì lỗi do gọi trực tiếp macro. Mục 9-1 có một
số mã cố gắng truy cập một chỉ mục trong một vector nằm ngoài phạm vi của các chỉ mục hợp lệ.

<span class="filename">Filename: src/main.rs</span>

```rust,should_panic,panics
{{#rustdoc_include ../listings/ch09-error-handling/listing-09-01/src/main.rs}}
```

<span class="caption">Mục 9-1: Cố truy cập một phần tử ngoài phạm vi của một vector, và sẽ dẫn đến lời gọi `panic!`</span>

Ở đây, chúng ta đang cố gắng truy cập phần tử thứ 100 của vector (tại chỉ mục
số 99 vì chỉ mục bắt đầu từ 0), nhưng vector chỉ có 3 phần tử. Trong tình huống này,
Rust sẽ cho ra lỗi. Sử dụng `[]` được cho là để trả về một phần tử, nhưng nếu bạn chuyển
vào một chỉ mục không hợp lệ, thì không có phần tử hợp lệ nào mà Rust có thể trả về ở đây.

Trong C, cố gắng đọc vượt quá phần cuối của cấu trúc dữ liệu là hành vi không
xác định. Bạn có thể nhận được bất kỳ thứ gì ở vị trí trong bộ nhớ tương ứng với
phần tử đó trong cấu trúc dữ liệu, mặc dù bộ nhớ không thuộc về cấu trúc đó.
Đây được gọi là *ghi đè bộ đệm* và có thể dẫn đến lỗ hổng bảo mật nếu kẻ tấn công
có thể thao túng chỉ mục, theo cái cách đọc dữ liệu mà chúng không được phép,
được lưu trữ phía sau cấu trúc dữ liệu.

Để bảo vệ chương trình của bạn khỏi loại lỗ hổng này, nếu bạn cố gắng đọc
một phần tử tại một chỉ mục không tồn tại, Rust sẽ ngừng thực thi và từ chối
việc tiếp tục. Hãy thử nó và xem như thế nào:

```console
{{#include ../listings/ch09-error-handling/listing-09-01/output.txt}}
```

Lỗi này trỏ đến dòng 4 của `main.rs` nơi chúng ta cố gắng truy cập chỉ mục 99.
Dòng lưu ý tiếp theo cho chúng ta biết rằng, có thể đặt biến môi trường `RUST_BACKTRACE`
để lấy dấu vết chính xác những gì xảy ra đã gây ra lỗi. *Backtrace* là danh sách tất cả
các hàm đã được gọi đến thời điểm này. Truy vết trong Rust hoạt động giống như các ngôn
ngữ khác: chìa khóa để đọc dấu vết là bắt đầu từ trên cùng và đọc cho đến khi bạn nhìn
thấy tập tin bạn đã viết. Đó là nơi khởi nguồn của vấn đề. Các dòng phía trên vị trí
đó là mã mà đoạn mã của bạn gọi tới; các dòng bên dưới là mã đã gọi đoạn mã của bạn.
Các dòng trước và sau có thể bao gồm mã trong lõi của Rust, mã thư viện chuẩn hoặc các
thư viện (crate) mà bạn đang sử dụng. Hãy thử truy vết bằng cách đặt biến môi trường
`RUST_BACKTRACE` thành bất kỳ giá trị nào ngoại trừ 0. Mục 9-2 hiển thị đầu ra tương
tự như những gì bạn sẽ thấy.

<!-- manual-regeneration
cd listings/ch09-error-handling/listing-09-01
RUST_BACKTRACE=1 cargo run
copy the backtrace output below
check the backtrace number mentioned in the text below the listing
-->

```console
$ RUST_BACKTRACE=1 cargo run
thread 'main' panicked at 'index out of bounds: the len is 3 but the index is 99', src/main.rs:4:5
stack backtrace:
   0: rust_begin_unwind
             at /rustc/7eac88abb2e57e752f3302f02be5f3ce3d7adfb4/library/std/src/panicking.rs:483
   1: core::panicking::panic_fmt
             at /rustc/7eac88abb2e57e752f3302f02be5f3ce3d7adfb4/library/core/src/panicking.rs:85
   2: core::panicking::panic_bounds_check
             at /rustc/7eac88abb2e57e752f3302f02be5f3ce3d7adfb4/library/core/src/panicking.rs:62
   3: <usize as core::slice::index::SliceIndex<[T]>>::index
             at /rustc/7eac88abb2e57e752f3302f02be5f3ce3d7adfb4/library/core/src/slice/index.rs:255
   4: core::slice::index::<impl core::ops::index::Index<I> for [T]>::index
             at /rustc/7eac88abb2e57e752f3302f02be5f3ce3d7adfb4/library/core/src/slice/index.rs:15
   5: <alloc::vec::Vec<T> as core::ops::index::Index<I>>::index
             at /rustc/7eac88abb2e57e752f3302f02be5f3ce3d7adfb4/library/alloc/src/vec.rs:1982
   6: panic::main
             at ./src/main.rs:4
   7: core::ops::function::FnOnce::call_once
             at /rustc/7eac88abb2e57e752f3302f02be5f3ce3d7adfb4/library/core/src/ops/function.rs:227
note: Some details are omitted, run with `RUST_BACKTRACE=full` for a verbose backtrace.
```

<span class="caption">Mục 9-2: dấu vết được tạo ra bởi lời gọi `panic!` được hiển thị khi biến môi trường` RUST_BACKTRACE` được thiết lập</span>

Có rất nhiều thông tin đầu ra! Kết quả chính xác mà bạn thấy có thể khác nhau tùy thuộc
vào hệ điều hành và phiên bản Rust của bạn. Để có được các dấu vết với thông tin này,
các biểu tượng gỡ lỗi phải được bật. Các biểu tượng gỡ lỗi được bật theo mặc định
khi sử dụng `cargo build` hoặc `cargo run` mà không có cờ `--release`,
như chúng ta chạy ở đây.

Trong đầu ra của mục 9-2, dòng 6 của dấu vết trỏ đến dòng trong dự án của chúng ta
đang gây ra sự cố: dòng 4 của *src/main.rs*. Nếu chúng ta không muốn chương trình
của mình bị lỗi, chúng ta nên bắt đầu điều tra tại vị trí được chỉ ra bởi dòng
đầu tiên đang đề cập đến một tệp chúng ta đã viết. Trong mục 9-1, nơi chúng ta
cố tình viết mã có thể gây lỗi, cách để khắc phục tình trạng gây lỗi này là không
yêu cầu một phần tử nằm ngoài phạm vi của chỉ mục vector. Khi mã của bạn bị lỗi trong
tương lai, bạn sẽ cần tìm ra hành động mà mã đang thực hiện với những giá trị nào
gây ra lỗi và mã nên làm gì thay thế.

Chúng ta sẽ quay lại với `panic!`, khi nào chúng ta nên và không nên sử dụng `panic!`
để xử lý điều kiện lỗi trong phần [`panic!` hoặc Không `panic!`”][to-panic-or-not-to-panic]<!-- ignore --> ở phần sau của chương này. Kế đến, chúng ta sẽ xem xét cách làm sao để
khôi phục lỗi bằng cách sử dụng `Result`.

[to-panic-or-not-to-panic]:
ch09-03-to-panic-or-not-to-panic.html#to-panic-or-not-to-panic
