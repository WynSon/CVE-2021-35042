#            CVE-2021-35042: Django SQL injection vulnerability

## I. Tổng quan

Django là một Web Application Framework mã nguồn mở, được viết bằng python được xây dựng theo mô hình MVC ( Model - View - Controller). Ban đầu nó được xây dựng để quản lý các trang web nội dung tin tức thuộc sở hữu bởi tập đoàn xuất bản Lawrence, phần mềm CMS (Content Management System).

Django version 3.1.x -> 3.1.13 và version 3.2.x -> 3.2.5 tồn tại lỗ hổng SQL injection.

Nguyên nhân dẫn tới lỗ hổng này bởi vì chức năng filter dữ liệu đầu vào do người dùng kiểm soát tại QuerySet.order_by() không đủ để phòng chống lại các cuộc tấn công SQL injection. Lỗ hổng này có thể bị khai thác cho phép kẻ tấn công thực hiện các hành động trái phép dẫn đến việc dò dỉ dữ liệu nhạy cảm.

| CVE - ID  | CVE-2021-35042  |
|---|---|
| **Severity**  | 9.8 - CRITICAL |
| **CWE - ID**  | CWE-89: Improper Neutralization of Special Elements used in an SQL Command ('SQL Injection')  |
| **Vulnerability Publication Date**  | 1/7/2021  |
| **Affected Software**  | 3.1.x < 3.1.13, 3.2.x < 3.2.5   |
| Require Authentication| No required|


## II. Tổng Quan x2

### 0x01. Django's Model

Trong django, việc tạo bảng và định nghĩa các trường trong database được thực hiện bằng cách khai báo một lớp model trong file `models.py`. Trong ví dụ này, ta khai báo một bảng có tên **Wolf**. và một trường có tên là ***name***.
![](https://i.imgur.com/P1h0vda.png)![](https://i.imgur.com/2QiqrWT.png)


### 0x02. QuerySet và Order_by() trong django

ORM framework tích hợp sẵn trong Django được sử dụng để thao tác với cở sở dữ liệu, và kết quả  của câu truy vấn là một môt tập hợp, tập hợp này là một ***QuerySet***.

***order_by(fields)***
Theo mặc định, `order_by()` kết quả trả về một QuerySet được sắp xếp theo một thứ tự được chỉ định trong tùy trọn `ordering` trong Model's **Meta**. Chúng ta có thể ghi đè điều kiện order_by trong mỗi truy vấn bằng việc sử dụng phương thức ` order_by()`.

> Ví dụ
>> wolves = Wolf.objects.order_by('-name', 'id')
>> 
Kết quả của câu truy vấn trên sẽ được sắp xếp giảm dần theo trường ***name***, sau đó tăng dần theo ***id***. Dấu âm ở trước tên trường ***name*** chỉ ra rằng sắp xếp kết quả giảm dần.

> Ví dụ sau sẽ sắp xếp kết quả trả về theo trường được nhận vào từ người dùng, nếu không có giá trị nào được truyền vào thì sẽ sắp xếp theo trường `id`.
> ![](https://i.imgur.com/Y95Cg8n.png)

`Kết quả`
                  ![](https://i.imgur.com/z0Q6X2S.png)

Tại phiên bản 3.1 và 3.2, Django cho phép kết hợp phương thức truy vấn với tên bảng vào câu truy vấn **order_by**. Đây cũng là nguyên nhân chính dẫn tới lỗ hổng này.

> Việc truyền vào một tên bảng cho chúng ta cùng một kết quả với việc truyền tên trường như bình thường
> ![](https://i.imgur.com/f6LOlB4.png)
>> cve202135042_wolf là tên bảng

Đầu tiên ứng dụng gọi trực tiếp tới hàm order_by(), đoạn mã xử lý hàm order _by() được định nghĩa tại: 
`django/db/models/query.py`
![](https://i.imgur.com/gsjMuPn.png)
> Hàm order_by() thực hiện 2 việc
>> ![](https://i.imgur.com/g3tiB3a.png)
>> 1. Xóa tất cả các phương hiện tại đang được gọi bởi order_by() và xóa tham số mặc định được truyền vào khi order_by nhận một giá trị khác.

>> 2. Truyền tham số vào order_by. Hàm `add_ordering()` sẽ thực hiện việc này
```python=
def add_ordering(self, *ordering):
        """
        Add items from the 'ordering' sequence to the query's "order by"
        clause. These items are either field names (not column names) --
        possibly with a direction prefix ('-' or '?') -- or OrderBy
        expressions.

        If 'ordering' is empty, clear all ordering from the query.
        """
        errors = []
        for item in ordering:
            if isinstance(item, str):
                if '.' in item:
                    warnings.warn(
                        'Passing column raw column aliases to order_by() is '
                        'deprecated. Wrap %r in a RawSQL expression before '
                        'passing it to order_by().' % item,
                        category=RemovedInDjango40Warning,
                        stacklevel=3,
                    )
                    continue
                if item == '?':
                    continue
                if item.startswith('-'):
                    item = item[1:]
                if item in self.annotations:
                    continue
                if self.extra and item in self.extra:
                    continue
                # names_to_path() validates the lookup. A descriptive
                # FieldError will be raise if it's not.
                self.names_to_path(item.split(LOOKUP_SEP), self.model._meta)
            elif not hasattr(item, 'resolve_expression'):
                errors.append(item)
            if getattr(item, 'contains_aggregate', False):
                raise FieldError(
                    'Using an aggregate in order_by() without also including '
                    'it in annotate() is not allowed: %s' % item
                )
        if errors:
            raise FieldError('Invalid order_by arguments: %s' % errors)
        if ordering:
            self.order_by += ordering
        else:
            self.default_ordering = False
            
```

Tham số được truyền vào `add_ordering()` là một mảng.
> Ví dụ khi tham số được truyền vào như sau:
> `wolves  =  Wolf.objects.order_by( 'name' ,  'id' )`
> Khi đó, ứng dụng sẽ thực hiện chuyển đổi thành câu truy vấn trong CSDL như sau:
```sql=
SELECT "cve202135042_wolf"."id", "cve202135042_wolf"."name" FROM "cve202135042_wolf" ORDER BY "cve202135042_wolf"."name" ASC, "cve202135042_wolf"."id" ASC
```


Khi được truyền vào, hàm add_ordering sẽ thực hiện kiểm tra từng phần tử trong mảng, nếu nó là một `string`, nó sẽ được kiểm tra 5 trường hợp sau:
> 1. `if  '.' in item:`
> Nó sẽ kiểm tra iệu rằng đó là một truy vấn có tên cột và cột đó có tên bảng được chỉ định trong câu lệnh SQL hay không. Nếu có thì sẽ đưa ra một cảnh báo và `continue`.
> 2. `if item == '?':`
> Nếu giá trị của phần tử là dấu '?', kết quả đầu ra sẽ được sắp xếp theo ngẫu nhiên, `continue`.
> 3. `if item.startswith('-'):`
> Nếu item bắt đầu với kí tự '-, kết quả của câu truy vấn sẽ được sắp xếp DESC(Giảm dần).
> 4. `if item in self.annotations:`
> Nó kiểm tra xem có chứa một comment hay không, nếu có, `continue`.
> 5. `if self.extra and item in self.extra:`
> Xác định xem có thêm bổ sung và nếu có thì `continue`.

Sau 5 lần kiểm tra, tham số tiếp tục được truyền vào hàm `self.names_to_path(item.split(LOOKUP_SEP), self.model._meta)` để tiếp tục kiểm tra xem liệu rằng nó có phải là một tên cột hợp lệ hay không, sau đó nếu hợp lệ thì chúng sẽ được thêm vào `self.ordering` của lớp `Query` để xử lý tiếp .

## III. Phân tích lỗ hổng SQL injection trong Django

### 0x31. Nguyên nhân 

Django's ORM thực hiện việc filter các dữ liệu được đưa vào câu truy vấn một cách rất nghiêm ngặt, nhưng việc thay đổi mã nguồn lần này dẫn đến SQL injection là do tác giả đã đưa ra giả thuyết rằng nếu tên cột là một [UUID](https://www.postgresqltutorial.com/postgresql-tutorial/postgresql-uuid/) (Universal Unique Identifer) column thì câu truy vấn `order_by` sẽ không thể được thực hiện. 

Nghĩa là nếu dữ liệu truyền vào là `xxx-xxx-xxx-xxx` (định dạng của UUID), câu truy vấn sẽ không thể thực hiện.

***Đoạn mã trước khi thực hiện thay đổi***
```python=
# django/db/models/sql/constants.py 
ORDER_PATTERN  =  _lazy_re_compile ( r '\?|[-+]?[.\w]+$' )

# django/db/models/sql/query.py 
def  add_ordering ( self ,  * ordering ): 
        errors  =  [] 
        for  item  in  ordering : 
            if  isinstance ( item ,  str )  and  ORDER_PATTERN . match ( item ): 
                if  '.'  in  item : 
                    warnings . warn ( 
                        'Passing column raw column aliases to order_by() is ' 
                        'deprecated. Wrap %r in a RawSQL expression before '
                        'passing it to order_by().'  %  item , 
                        category = RemovedInDjango40Warning , 
                        stacklevel = 3 , 
                    ) 
            elif  not  hasattr ( item ,  'resolve_expression' ): 
                errors . append ( item ) 
            if  getattr ( item ,  'contains_aggregate' ,  False ): 
                raise  FieldError ( 
                    'Using an aggregate in order_by() without also including ' 
                    'it in annotate() is not allowed: %s ' %  item 
                ) 
        if  errors : 
            raise  FieldError ( 'Invalid order_by arguments: %s '  %  errors ) 
        if  ordering : 
            self . order_by  +=  ordering 
        else : 
            self . default_ordering  =  False
```
Từ đoạn mã trên, ta có thể thấy rằng nếu tham số match với `?` hoặc được bắt đầu bằng `-` và theo sau đó là các kí tự hoặc dấu `.` thông thường thì câu truy vấn mới được thực hiện. 

Do đó, khi tên cột là một UUID thì nó sẽ là một giá trị không hợp lệ và không thể đưa vào `order_by`.
Việc thay đổi code phần xử lý này đã được chấp nhận, nó được thay đổi như sau:
https://github.com/charettes/django/commit/513948735b799239f3ef8c89397592445e1a0cd5

![](https://i.imgur.com/XqX3bqA.png)

Nó đã sử dụng hàm `self.name_to_path` để xác thực dữ liệu đầu vào. 

Nhưng sau khi kiểm tra nếu `.` có trong item, nó sẽ coi đó là một truy vấn với tên bảng, lệnh `continue` được thực thi, dẫn tới việc trực tiếp bỏ qua việc sử dụng hàm `self.name_to_path` để xác kiểm tra tính hợp lệ của dữ liệu.

Đoạn code xử lý dấu `.` tại hàm `get_order_by` như sau
`django/db/models/sql/compiler.py`

```python=
if  '.'  in  field : 
    table ,  col  =  col . split ( '.' ,  1 ) 
    order_by . append (( 
            OrderBy ( 
                RawSQL ( ' %s . %s '  %  ( 
                self . quote_name_unless_alias ( table ),  col ),  [ ]), 
                descending = descending 
            ),  False )) 
    continue
```
Hàm `self.quote_name_unless_alias` xử lý tên bảng, lọc các tên bảng hợp lệ và bỏ qua việc filter tên cột, do đó ta có thể chèn một câu lệnh SQL injection.

### 0x32. Bản Vá

Phiên bản Django 4.0 hiện tại, việc truy vấn theo tên bảng bằng dấu `.` đã bị loại bỏ và không còn hỗ trợ nữa, bản vá được đưa ra cho phiên bản 3.1 và 3.2. Các phiên bản 3.2 -> 3.2.4 và 3.1 -> 3.1.12 bị ảnh hưởng.
[3.2.x Fixed CVE-2021-35042 -- Prevented SQL injection in QuerySet.o… ](https://github.com/django/django/commit/a34a5f724c5d5adb2109374ba3989ebb7b11f81f#diff-fd2300283d1546e36141373b0621f142ed871e3e8856e07efe5a22ecc38ad620)

[3.1.x Fixed CVE-2021-35042 -- Prevented SQL injection in QuerySet.o…](https://github.com/django/django/commit/0bd57a879a0d54920bb9038a732645fb917040e9)

Việc sửa đổi rất đơn giản, việc kiểm tra dữ liệu bằng ReGex cũ đã được đưa trở lại
![](https://i.imgur.com/yM9Zzi9.png)

### 0x33. Khắc Phục

***Cập nhật Django lên phiên bản không bị ảnh hưởng.***

## IV. Demo

### 0x41. Môi trường

**Docker & Docker-compose**

### 0x42. Setup


1. `git clone https://github.com/WynSon/CVE-2021-35042.git`
2. Run `./setup.sh` for initial setup
3. `sudo docker-compose up --build`
4. `sudo docker exec -it cve-2021-35042_web_1 python manage.py makemigrations cve202135042`
5. `sudo docker exec -it cve-2021-35042_web_1 python manage.py migrate`
6. Truy cập http://localhost:8000/load_example_data để load sample data:
7. Đường dẫn chứa vulnerable param: http://localhost:8000/wolves/
`http://localhost:8000/wolves/?order_by=name`

Màn hình sau khi cài đặt xong

![](https://i.imgur.com/ja1HouJ.png)

### 0x43. Khai thác

**Điều kiện:** ***Để có thể khai thác được, chúng ta bắt buộc phải biết được tên bảng bằng 1 cách nào đó :))***

Khi inject câu lệnh, chúng ta phải biết được tên bảng thì mới có thể thực thi câu lệnh SQLi được.

***Khi nhập sai tên bảng***
![](https://i.imgur.com/067YxM0.png)



***Khi nhập đúng tên bảng, câu truy vấn orderby thực hiện bình thường.***
![](https://i.imgur.com/ozeFvbj.png)

**Câu lệnh lúc này sẽ thành**
`SELECT "cve202135042_wolf"."id", "cve202135042_wolf"."name" FROM "cve202135042_wolf" ORDER BY ("cve202135042_wolf"."name") ASC
`


Lúc này, ta có thể kết thúc câu lênh order_by phía trước và chèn một câu lệnh SQL vào để khai thác.

![](https://i.imgur.com/sbkmrS0.png)

`SELECT "cve202135042_wolf"."id", "cve202135042_wolf"."name" FROM "cve202135042_wolf" ORDER BY ("cve202135042_wolf"."name"); SELECT * from cve202135042_wolf where id =1; --) ASC`


## V. Reference

https://www.djangoproject.com/weblog/2021/jul/01/security-releases/
https://xz.aliyun.com/t/9834
https://www.bugxss.com/vulnerability-report/3095.html
https://blankheart.top/2022/04/07/cve-2021-35042/
https://itcn.blog/p/1648921763575859.html
https://github.com/YouGina/CVE-2021-35042