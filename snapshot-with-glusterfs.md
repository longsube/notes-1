###Cấu hình GlusterFS để sử dụng tính năng snapshot trên GlusterFS
####Thinly provisioned LVs

Bài viết sau đây sẽ hướng dẫn làm thế nào để sử dụng tính năng snapshot trên glusterfs.


Thử sử dụng tính năng này khi chưa cấu hình gì thì sẽ có thông báo sau

<img src="http://i.imgur.com/TnZy75R.png">

Snapshot chỉ hỗ trợ cho thin provisioned LV. Do vậy chúng ta sẽ cấu hình thin provisioned LV.

- Tất cả các bricks sẽ độc lập trên một logical volume (thinly provisioned logical volume). Nói cách khác, không có 2 brick cùng chia sẻ chung một LV.

- thinly provisioned LV này chỉ nên được sử dụng cho 1 bricks

- các thin LV sẽ được tạo trên thin pool nên có đủ không gian chứa cho các thin LV và metadata của pool.

Hình sau biểu diễn vị trí thin pool, thin LV và volume group

<img src="http://i.imgur.com/CbAVUdK.png">

Hình trên đưa ra một cách tổng quát của LVM.
- Volume group được tạo thành từ các thiết bị lưu trữ (ổ cứng).
- Một hoặc nhiều thin pool có thể được tạo trên volume group này
- Sau đó ta có thể tạo một hoặc nhiều thinly provisioned logical volume (LV).
- Tất cả các LV được tạo với 1 kích thước ảo. Tổng kích thước này thậm chí còn lớn hơn dung lượng của pool. Ví dụ bạn có 1 ổ cứng 1TB nhưng ta có thể tạo ra tổng dung lượng các LV là 4TB. Vì vậy bạn có thể thêm ổ khi cần thiết.

##### Cấu hình
- ở bài lab này mình sẽ gắn thêm 1 ổ cứng có dung lượnng 20GB vào node đang chạy glusterfs-server.

- Tạo physical volume

```
pvcreate /dev/sdb

```

- Tạo volume group đặt tên à `glusterfs`

```

vgcreate glusterfs /dev/sdb

```

- Sau đó ta có thể tạo thin pool `mythinpool` trên volume group `glusterfs`

```
lvcreate -L 2G -T glusterfs/mythinpool

```

- Tạo thin provisioned LV `vol1` trong pool trên

```
lvcreate -V 1G -T glusterfs/mythinpool -n vol1

```

- Cuối cùng ta thực hiện tạo file-system trên LV này và mount.  xfs ext4 được recommend

```
mkfs.xfs /dev/glusterfs/vol1
mkdir -p /gluster/vol1

mount /dev/glusterfs/vol1 /gluster/vol1

```

#####Tạo thin provisioned logical volume

<img src="http://i.imgur.com/WgVdg3Z.png">


- Bây giờ ta có thể tạo volume trên thin provisioned LV này.

```
gluster volume create thin-vol1 glusterfs:/gluster/vol1/br1
gluster volume start thin-vol1

```

<img src="http://i.imgur.com/ixe3CGy.png">


- Tạo thử snapshot

```
gluster snapshot create snap-vol1 thin-vol1

```

<img src="http://i.imgur.com/ixe3CGy.png">




- Kiểm tra danh sách snapshot trên `thin-vol1`

```
gluster snapshot list thin-vol1

```

<img src="http://i.imgur.com/4BvZfZg.png">


Như vậy ta đã cấu hình thành công thin provisioned LV vào tính năng snapshot trên GlusterFS đã có thể sử dụng được.
