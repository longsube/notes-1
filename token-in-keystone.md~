###Bốn kiểu token của Keystone trong Openstack
Token là một dạng thông tin của một user, token được sinh ra khi ta sử dụng username,password đúng để xác thực với keystone. Khi đó user sẽ dùng token này để truy cập vào Openstack API.

<img src="http://i.imgur.com/bvN0CyI.png">

###Hiện tại Openstack Keystone hỗ trợ 4 kiểu token.
Ở phiên bản Diablo chỉ sử dụng kiểu UUID. Với dạng token này thì rất dễ gặp vấn đề về hiệu năng của Keystone. Bất cứ khi nào User truy cập vào một Openstack API thì service đó sẽ phải mang token cho Keystone xác thực lại. Nếu hệ thống mở rộng lớn, lượng request lớn do đó keystone càng phải làm việc nhiều, trong khi đó token lưu tại database sẽ tăng kích thước rất nhanh, Ảnh hưởng tới hiệu năng của toàn bộ hệ thống.

Vì vậy PKI Token ở phiên bản Grizzly đã được đưa vào sử dụng. Với việc dùng PKI Token các thông tin về user (user, project, role, ...) sẽ được keystone mã hóa bằng Private Key. Khi user sử dụng token này với Openstack API service sẽ sử dụng Public Key để giải mã lấy các thông tin. Khi hệ thống mở rộng các thông tin về user càng nhiều thì kích thước của token cũng tăng theo. Mà header của HTTP giới hạn 8KB nếu vượt quá giới hạn này request sẽ bị lỗi.

Vì vấn đề này,Openstack đã phát triển PKIZ Token với việc nén token làm kích thước của token giảm xuống. Nhưng việc nén là có hạn chế do vậy không thể giải quyết được vấn đề với Token quá lớn.

Phía trên là 3 kiểu token persistent do vậy token sẽ được lưu trong database. Khi hệ thống sử dụng lâu dài việc xóa bỏ token là việc thường xuyên của Operator. Để giải quyết vấn đề trên cộng đồng Openstack đã phát triển Fernet token ở phiên bản Kilo trở đi. Và khi Mitaka phát hành họ đã sử dụng Fernet Token làm mặc định. Với Fernet token sẽ chứa một ít thông tin về user, có kích thước khoảng 255 Bytes và không lưu trong database.

####UUID
Token được fixed độ dài 32Bytes là một chuỗi hex được sinh random.

Ví dụ về một token UUID: 
```
144d8a99a42447379ac37f78bf0ef608
```

UUID không mang thông tin gì về User. Keystone phải thực hiện việc xác thực token và lưu trữ, với kích thước của hệ thống mở rộng hơn thì hiệu xuất của Keystone sẽ bị ảnh hưởng.

####PKI

<img src="http://i.imgur.com/PJxQma4.png">

PKI Token bản chất dựa trên chữ ký điện tử. Keystone sẽ dùng private key cho việc ký nhận và các Openstack API sẽ có public key để xác nhận thông tin đó.

####PKIZ

<img src="http://i.imgur.com/XDKJwNz.png">


PKIZ dựa trên PKI nhưng sẽ được nén lại làm giảm kích thước của token.

####Fernet

<img src="http://i.imgur.com/3ITT8xg.png">



Để giải quyết một số vấn đề trên thì cộng đồng Openstack đã phát triển Fernet token. Nó sử dụng một thư viện mã hoá cân xứng (Mã hóa, Giải mã sử dụng chung key) mã hóa token (AES-CBC và SHA256). Fernet được thiết kế cho Token nhẹ, bảo mật thông tin, không cần lưu trữ trong database làm giảm IO của ổ đĩa, cải thiện hiệu năng. Để cải thiện vấn đề bảo mật, ta sử dụng kỹ thuật Key Rotation để thay thế key.

Fernet token sẽ bao gồm các thông tin: userid, projectid, domainid, methods, expiresat, và các thông tin khác. Quan trọng, nó không bao gồm service_catalog vì vậy khi region tăng lên thì không ảnh hưởng tới kích thước của token.

Một ví dụ về fernet token:

```
gAAAAABWfX8riU57aj0tkWdoIL6UdbViV-632pv0rw4zk9igCZXgC-sKwhVuVb-wyMVC9e5TFc  
7uPfKwNlT6cnzLalb3Hj0K3bc1X9ZXhde9C2ghsSfVuudMhfR8rThNBnh55RzOB8YTyBnl9MoQ  
XBO5UIFvC7wLTh_2klihb6hKuUqB6Sj3i_8  
```

Chọn token phù hợp.
Bên dưới là bảng so sánh 4 kiểu token với các thông số chính ta có thể chọn loại token phù hợp với hệ thống của mình.

<img src="http://i.imgur.com/gnH4VRJ.png">
