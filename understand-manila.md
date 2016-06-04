###Tổng quan dịch vụ Shared File System

Dịch vụ Openstack Shared File Systems (manila) cung cấp file storage cho một máy ảo. Dịch vụ Shared File Systems cung cấp một cơ sở hạ tầng để quản lý và cung cấp các file chia sẻ. Dịch vụ cũng cho phép quản lý các kiểu chia sẻ cũng như các share snapshot nếu driver hỗ trợ chúng.

Dịch vụ Shared File System có các thành phần sau.

- manila-api: xác thực và route các request tới dịch vụ. Nó hỗ trợ các API Openstack.

- manila-scheduler: Lập lịch và chuyển các requét với share service thích hợp. Scheduler sử dụng các cấu hình lọc và weigh để route các requét. Filter Scheduler là  mặc định và cho phép filter dựa trên: Khả năng lưu trữ (Capacity), Các zone (Availability Zone), Kiểu share (share-type) và các filter custom.

- manila-share: quản lý các backend cung cấp shared file system. Một tiến trình manila-share có thể chạy trên 1 hoặc 2 node.

####Share, Snapshot,share-network

Là các resource cơ bản đưa ra bởi dịch vụ Shared File Systems là shares, snapshots, và share network

- share: là một đơn vụ của storage với một giao thức, một kích thước và một danh sách được phép truy cập. Tất cả các share tồn tại trên một backend. Một vài share tương tác với share network và share server. Các giao thức chính được hỗ trợ là NFS và CIFS, ngoài ra còn có các giao thức khác được hỗ trợ như GLUSTERFS, ..

- snapshots:  Snapshot là một thời điểm copy của một share. Snapshot có thể được sử dụng để tạo một share mới. Share không thể bị xóa nếu có các snapshot tạo ra trên share đó(ta phải xóa hết snapshot của share đó  rồi mới xóa được share)

-share network: Một share network là một đối được được định nghĩa bởi một project(tenant) mà báo cho Manila về security và cấu hình mạng cho một nhóm của các share. Share network chỉ thích hợp cho backend quản lý share server.  Một share network bao gồm các dịch vụ bảo mật , network và subnet.

####Cài đặt manila

Với manila ta có thể cấu hình trên một node hoặc nhiều node. Mình sẽ cài đặt trên nhiều node
- Các thành phần manila-api và manila-scheduler ta sẽ cài đặt trên node controller
- Và manila-share sẽ cài đặt trên một node riêng.

Khi cài đặt manila sẽ cho ta hai lựa chọn. Đó là cấu hình có hoặc không có việc xử lý của share server. Và các lựa chọn này dựa trên driver ta sử dụng.

- Lựa chọn 1: Cấu hình không có việc xử lý của share server. Trong mode này dịch vụ không can thiệp tới network. Người vận hành phải chắc chắn việc kết nối nữa các instance và Share Server. Với lựa chọn này ta sử dụng driver LVM, GLUSTERFS,...

- Lựa chọn 2: Triển khai dịch vụ vói sự hỗ trợ của driver cho việc quản lý các share. Trong mode này, các service yêu càu là nova, neutron và cinder cho việc quản lý share server. Thông tin được sử dụng cho việc tạo share server là share network. Option này sử dụng generic driver với việc sử lý các  share server và đòi hỏi phải được gắn vào mạng `selfservice` tới một router.

Với lựa chọn 2 ta có thể hình dung đơn giản như sau.
Để sử dụng option này ta cần phải tạo ra một share-network dựa trên cấu hình neutron.
Share network này sẽ được gắn với router của tenant và thông với tenant-network.
Sau khi có một yêu cầu tạo share. Khi Yêu cầu này sẽ được chuyển tới 1 backend có Cấu hình `Generic Driver` trên node cài manila-share. manila-share được cấu hình sử dụng nova để tạo ra một instance (image được cài sẵn nfs-server), đồng thời 1 volume tạo ra trên cinder có kích thước bằng kích thước của share tạo ra volume này gắn với instance trên. Và các dữ liệu share sẽ được lưu tại volume này.

Mô hình mạng sẽ như sau

<img src="http://i.imgur.com/TmiBQi7.png">

Notes: Mình mới test thử trên Openstack L2-Agent sử dụng OpenvSwitch chưa test được trên LinuxBridge

####Cấu hình một số driver share

- Các cấu hình dưới đây được thực hiện trên Ubuntu 14.04
- Riêng Ganesha-server được cài trên Centos 7.2

#####LVM

- Với backend là LVM ta sẽ cần tạo ra một Volume Group và khai báo Volume Group này cho manila-share

- Trên node cài manila-share ta cần cài thêm gói lvm2 và nfs-kernel-server

```
 apt-get install lvm2 nfs-kernel-server -y

```

- Mình sẽ add thêm một ổ cứng để tạo Volume Group trên ổ cứng này.

- Thực hiện tạo volume group
```
pvcreate /dev/sdb
vgcreate manila-volumes /dev/sdb

```

<img src="">

- Đặt filter cho LVM. Bởi vì mặc định LVM sẽ scan trong thư mục /dev của  các thiết bị block storage device mà bao gồm các volume. Nếu các project sử dụng LVM trên những volume của họ, tool scan sẽ phát hiện những volume đó và cố gắng để cache chúng, do vậy nó có thể gây ra nhiều vấn đề với cả OS và project volume. Cấu hình để LVM chỉ scan ổ có manila-volumes volume group.

Mở file /etc/lvm/lvm.conf tìm dòng sau

```
filter = ["a/sdb","r/.*a/"]

```

*Với a là allow còn r là reject với cấu hình này ta sẽ chỉ cho phép scan trên ổ sdb*

- Cấu hình manila-share. Ta mở file /etc/manila/manila.conf và cấu hình như sau

Thêm vào phần [DEFAULT] như sau

```
enabled_share_backends = lvm1  # lvm1 là tên backend
enabled_share_protocols = NFS,CIFS # giao thức cho phép là NFS và CIFS

```

Tạo backend lvm1, ta thêm một phần [lvm1] như sau

```
[lvm1]
share_backend_name = LVM-1 # tên backend
share_driver = manila.share.drivers.lvm.LVMShareDriver # driver sử dụng
driver_handles_share_servers = False # False: driver không xử lý share server
lvm_share_volume_group = manila-volumes  ## tên volume group vừa tạo ở trên
lvm_share_export_ip = 172.16.25.134 ## IP sẽ expose cho các instance kết nối đến. mình sẽ dùng IP external do vậy các instance có     #thể kết nối đến được

```
- Restart manila-share

```
service manila-share restart

```

- Ta kiểm tra service manila-share đã up chưa bằng lệnh

```
manila service-list
```

- Tạo share-type DHSS = False nếu chưa tạo

```

manila type-create lvm False

```

- Để chỉ định 1 backend trên share-type nào đó ta thực hiện như sau

```
manila type-key lvm set share_backend_name=lvm1
```

- Tạo một share


```
manila create nfs 4 --name test-share-1 --share-type lvm

```
        - NFS là protocol
        - 4 là kích thước (GB)
        - `--name`: tên của share
        - `--share-type` chọn share-type
- Để list các share

```
manila list

```

- Sau khi tạo share ta cần một đường dẫn để mount trên client

```
manila show test-share-1

```

- Để client có thể mount được ta cần cấu hình access-allow. Với LVM sẽ là dựa trên địa chỉ IP, mặc định sẽ có quyền RW ta có thể thay đổi bằng thêm option `--access-level LEVEL` (RW, RO)


```
manila access-allow test-share-1 ip IP_CLIENT

```

- Để delete share

```
manila delete test-share-1
```
#####Generic Driver

- Yêu cầu khi sử dụng Generic Driver là bạn phải có cấu hình nova, neutron và cinder trong file cấu hình `/etc/manila/manila.conf` trên node manila-share

- Và trên node manila share sẽ cần cài đặt L2 Agent. ở đây mình sử dụng OpenvSwitch. Mình sẽ cài đặt manila-share trên node compute để không phải cấu hình lại openvswitch.

- Ta sẽ cần một image có sẵn share service. Tải image về từ openstack foundation

```
wget http://tarballs.openstack.org/manila-image-elements/images/manila-service-image-master.qcow2

```

- Upload image và tạo flavor.

```
openstack image create "manila-service-image" \
--file manila-service-image-master.qcow2 \
--disk-format qcow2 \
--container-format bare \
--public

openstack flavor create manila-service-flavor --id 100 --ram 256 --disk 0 --vcpus 1

```


- Thêm các mục sau vào `/etc/manila/manila.conf`. *Sửa lại password các user*

```
[nova]
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = saphi

[cinder]

auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = cinder
password = saphi

[neutron]

url = http://controller:9696
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = saphi

```

- Khai báo backend sử dụng, Thêm vào phần [DEFAULT] như sau

```
enabled_share_backends= generic1 #generic1 là tên phần backend
enabled_share_protocols=NFS,CIFS #giao thức cho phép
```

- Khai báo cấu hình backend

```
[generic1]
share_backend_name = GENERIC-1 # tên backend
share_driver = manila.share.drivers.generic.GenericShareDriver #driver được sử dụng là Generic
driver_handles_share_servers = True # True: Driver xử lý share servers
service_instance_flavor_id = 100 # khai báo id của flavor sử dụng khi launch share server
service_image_name = manila-service-image # image share server sử dụng
service_instance_user = manila # user đăng nhập share server
service_instance_password = manila # password đăng nhập share server
#interface_driver = manila.network.linux.interface.BridgeInterfaceDriver # nếu sử dụng LinuxBridge thì uncomment
interface_driver = manila.network.linux.interface.OVSInterfaceDriver # sử dụng driver cho OpenVswitch

```
- Reset lại dịch vụ manila-share

```
service manila-share restart
```

- Tương tự ta kiểm tra trạng thái các service manila

```
manila service-list
```

- Tạo share-type và share-network

```
manila type-create generic True

```
True ở đây có nghĩa là `Driver Handling Share Servers = True`

- Với Share-network ta sẽ cần net-id và subnet-id của project.

```
neutron net-list
```


#####Native GlusterFS Driver

- Yêu cầu phiên bản GlusterFS >= 3.6. Với glusterfs nếu cluster của bạn không hỗ trợ snapshot thì trên manila cũng sẽ mất đi tính năng này. Để cấu hình snapshot ta sẽ cấu hình Thin Provision theo bài hướng dẫn link

- Với bài lab của mình có 2 node và chạy kiểu replicate. Mình sẽ tạo các thinly provisioned và tạo volume trên đó.
Tham khảo 2 script sau.

Script tạo thinly provisioned chạy trên 2 node

```
apt-get install xfsprogs -y
pvcreate /dev/sdb
vgcreate myVG /dev/sdb
lvcreate -L 8G -T myVG/thinpool
for ((i = 1;i<= 5; i++ ))
do
mkdir -p /manila/manila-"$i"
for (( j = 1; j<= 5; j++))
do
lvcreate -V "${i}"Gb -T myVG/thinpool -n vol-"$i"-"$j"
mkfs.xfs /dev/myVG/vol-"$i"-"$j"
mkdir -p /manila/manila-"$i"/manila-"$j"
mount /dev/myVG/vol-"$i"-"$j" /manila/manila-"$i"/manila-"$j"
echo "/dev/myVG/vol-"$i"-"$j" /manila/manila-"$i"/manila-"$j" xfs 0 2" >> /etc/fstab
done
done


```

Tạo gluster volume chỉ cần chạy trên 1 node

```
for (( i= 1 ; i <= 5; i++))
do
for(( j= 1; j<=5 ;j++))
do
gluster volume create manila-"$i"-"$j" replica 2 glusterfs-1:/manila/manila-"$i"/manila-"$j"/br glusterfs-2:/manila/manila-"$i"/manila-"$j"/br
gluster volume start manila-"$i"-"$j"
done
done

```




- Cấu hình GlusterFS tương tự như LVM mình sẽ sử dụng node compute để cài manila-share.

- Trên node này ta cần cài glusterfs-client

######Cấu hình manila-share tại file `/etc/manila/manila.conf`

- enable backend glusterfsnative và protocol GLUSTERFS tại mục [DEFAULT]

```
enabled_share_backends= glusterfsnative #glusterfsnative là tên phần backend
enabled_share_protocols=NFS,CIFS,GLUSTERFS #giao thức cho phép

```
- Cấu hình backend glusterfsnative như sau

```
[glusterfsnative]
share_backend_name = glusterfsnative # tên backend
glusterfs_servers = root@glusterfs-1 # khai báo user,IP của gluster server, nếu có nhiều server glusterfs cách nhau bằng dấu phẩy
glusterfs_server_password = saphi # password để manila-host ssh vào glusterfs server
glusterfs_volume_pattern = manila-#{size}-.* # pattern volume trên manila khi tạo share mapping với volume trên glusterfs
share_driver = manila.share.drivers.glusterfs.glusterfs_native.GlusterfsNativeShareDriver # share driver sử dụng
driver_handles_share_servers = False  # False: không sử dụng driver xử lý share server
 
```

Nói thêm phần volume-pattern:

Với script ở trên mình đã tạo các volume có mẫu `manila-$i-$j` với $i là size của volume và $j là số thứ tự. Do vậy ta sẽ cấu hình mẫu để các share manila khi tạo ra sẽ mapping với các volume trên glusterfs.

- Sau khi cấu hình xong ta khởi động lại manila-share

```
service manila-share restart
```

- Kiểm tra service manila sẽ có thêm service cho glusterfsnative

- Tạo share-type

```
manila type-create glusterfsnative False # tạo type tên là glusterfsnative DHSS = False
manila type-key glusterfsnative set share_backend_name=glusterfsnative # khai báo backend cho type này
```

- Tạo share

```
manila create glusterfs 2 --name gluster-1 --share-type glusterfsnative
```

#####GlusterFS Driver with NFS-Ganesha làm gateway
