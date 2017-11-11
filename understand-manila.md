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



#####Generic Driver

- Yêu cầu khi sử dụng Generic Driver là bạn phải có cấu hình nova, neutron và cinder trong file cấu hình `/etc/manila/manila.conf` trên node manila-share

- Và trên node manila share sẽ cần cài đặt L2 Agent. ở đây mình sử dụng OpenvSwitch. Mình sẽ cài đặt manila-share trên node compute để không phải cấu hình lại openvswitch.

Mô hình cài đặt

<img src="http://i.imgur.com/D0Zk5uP.png">

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

- Kiểm tra image

```
openstack image list
```

<img src="http://i.imgur.com/xQSLoyY.png">

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

<img src="http://i.imgur.com/qf1e3sm.png">

- Tạo share-type và share-network

```
manila type-create generic True
```


True ở đây có nghĩa là `Driver Handling Share Servers = True`

<img src="http://i.imgur.com/MOt0QWR.png">

- Với Share-network ta sẽ cần net-id và subnet-id của project.

```
root@controller:/home/saphi# neutron net-list
+--------------------------------------+------------------------+------------------------------------------------------+
| id                                   | name                   | subnets                                              |
+--------------------------------------+------------------------+------------------------------------------------------+
| 85caa7b4-1212-4037-b350-5e2db386b416 | private_demo           | f5c151cd-5baa-4628-b8dc-43963115adc7 192.168.2.0/24  |
| 56fc4f98-ddc9-49a0-a014-5b86f7d99f82 | ext-net                | a1bb678f-d630-4c1a-8053-7af2676e2b94 172.16.25.0/24  |
| 9af94508-21a7-4c3f-b126-04adf719ec19 | private-net            | a6b97594-0338-4f99-beea-cd0d0d07363b 192.168.10.0/24 |
+--------------------------------------+------------------------+------------------------------------------------------ 
```
Mình đang ở project admin do vậy sẽ lấy subnet và net id của admin là dải `private-net`

```
root@controller:/home/saphi# manila share-network-create --name share-net-admin --neutron-subnet-id a6b97594-0338-4f99-beea-cd0d0d07363b --neutron-net-id 9af94508-21a7-4c3f-b126-04adf719ec19
+-------------------+--------------------------------------+
| Property          | Value                                |
+-------------------+--------------------------------------+
| name              | share-net-admin                      |
| segmentation_id   | None                                 |
| created_at        | 2016-06-05T11:02:17.209356           |
| neutron_subnet_id | a6b97594-0338-4f99-beea-cd0d0d07363b |
| updated_at        | None                                 |
| network_type      | None                                 |
| neutron_net_id    | 9af94508-21a7-4c3f-b126-04adf719ec19 |
| ip_version        | None                                 |
| nova_net_id       | None                                 |
| cidr              | None                                 |
| project_id        | 2dbc33e520b84b8ab550b649099d7972     |
| id                | 9a295209-63d7-4637-9efc-f6317e7bff8a |
| description       | None                                 |
+-------------------+--------------------------------------+
```

Kiểm tra share network đã tạo ra

<img src="http://i.imgur.com/XhbbA8Y.png">

Ta cũng thấy rằng sẽ có một mạng và subnet mới được tạo ra  với tên `manila_service_network `

<img src="http://i.imgur.com/j9cZc7i.png">


Thực hiện tạo share server sử dụng giao thức NFS và kích thước share 1GB có tên share-01

```
manila create nfs 1 --share-network share-net-admin --name share-01 
```

Và sau đó ta sẽ có share server

<img src="http://i.imgur.com/Ldol4lM.png">

Mình đang có một instance `sa` trên project admin có IP `192.168.10.7`
```
root@controller:/home/saphi# openstack server list
+--------------------------------------+------+--------+--------------------------+
| ID                                   | Name | Status | Networks                 |
+--------------------------------------+------+--------+--------------------------+
| dbf12e7f-1fec-4817-8a39-e7100835f092 | sa   | ACTIVE | private-net=192.168.10.7 |
+--------------------------------------+------+--------+--------------------------+
```
Tạo access-allow cho instance trên vào share `share-01`

<img src="http://i.imgur.com/NNSLOy8.png">

Để lấy đường dẫn mount `manila show share-01`

```
root@controller:/home/saphi# manila show share-01
+-----------------------------+-----------------------------------------------------------------------+
| Property                    | Value                                                                 |
+-----------------------------+-----------------------------------------------------------------------+
| status                      | available                                                             |
| share_type_name             | default_share_type                                                    |
| description                 | None                                                                  |
| availability_zone           | nova                                                                  |
| share_network_id            | 9a295209-63d7-4637-9efc-f6317e7bff8a                                  |
| export_locations            |                                                                       |
|                             | path = 10.254.0.12:/shares/share-d7287607-4750-4874-b3ca-bd9c734cee0e |
|                             | preferred = False                                                     |
|                             | is_admin_only = False                                                 |
|                             | id = 0db97dd9-0af2-49fb-907e-854263727995                             |
|                             | share_instance_id = d7287607-4750-4874-b3ca-bd9c734cee0e              |
| share_server_id             | 25c6e835-5dbe-4bbb-8451-29df0805aaef                                  |
| host                        | compute1@generic#GENERIC                                              |
| access_rules_status         | active                                                                |
| snapshot_id                 | None                                                                  |
| is_public                   | False                                                                 |
| task_state                  | None                                                                  |
| snapshot_support            | True                                                                  |
| id                          | fd6574cc-ac43-4f8d-bfd1-22edbf1bf113                                  |
| size                        | 1                                                                     |
| name                        | share-01                                                              |
| share_type                  | 4d7aca95-a7d0-4d66-9036-7e7e876d8c4c                                  |
| has_replicas                | False                                                                 |
| replication_type            | None                                                                  |
| created_at                  | 2016-06-05T11:06:50.000000                                            |
| share_proto                 | NFS                                                                   |
| consistency_group_id        | None                                                                  |
| source_cgsnapshot_member_id | None                                                                  |
| project_id                  | 2dbc33e520b84b8ab550b649099d7972                                      |
| metadata                    | {}                                                                    |
+-----------------------------+-----------------------------------------------------------------------+
```
Trên instance `sa` thưc hiện mount

<img src="http://i.imgur.com/mqCH11c.png">

#####LVM

Trên node cài manila-share ta cần cài thêm gói lvm2 và nfs-kernel-server. Do vậy manila-share sẽ trực tiếp quản lý các volume trên host này. Để mở rộng ta sẽ phải cài manila-share trên tất cả các node LVM.

Mô hình

<img src="http://i.imgur.com/4K8lnyR.png">

- Với backend là LVM ta sẽ cần tạo ra một Volume Group và khai báo Volume Group này cho manila-share

- Cài đặt các thành phần

```
add-apt-repository cloud-archive:mitaka
apt-get update
apt-get install manila-share python-pymysql lvm2 nfs-kernel-server -y
```

- Mình sẽ add thêm một ổ cứng để tạo Volume Group trên ổ cứng này.

- Thực hiện tạo volume group
```
pvcreate /dev/sdb
vgcreate manila-volumes /dev/sdb
```

<img src="http://i.imgur.com/0Xav3mA.png">

- Đặt filter cho LVM. Bởi vì mặc định LVM sẽ scan trong thư mục /dev của  các thiết bị block storage device mà bao gồm các volume. Nếu các project sử dụng LVM trên những volume của họ, tool scan sẽ phát hiện những volume đó và cố gắng để cache chúng, do vậy nó có thể gây ra nhiều vấn đề với cả OS và project volume. Cấu hình để LVM chỉ scan ổ có manila-volumes volume group.

Mở file `/etc/lvm/lvm.conf` tìm dòng sau

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
lvm_share_export_ip = 172.16.25.148 ## IP sẽ expose cho các instance kết nối đến. mình sẽ dùng IP external do vậy các instance có thể kết nối đến được
```
- Restart manila-share

```
service manila-share restart
```

- Ta kiểm tra service manila-share đã up chưa bằng lệnh

```
manila service-list
```
<img src="http://i.imgur.com/COhh0vb.png">

- Tạo share-type DHSS = False nếu chưa tạo

```
manila type-create lvm False
```

<img src="http://i.imgur.com/9jrTIxe.png">

- Để chỉ định 1 backend trên share-type nào đó ta thực hiện như sau

```
manila type-key lvm set share_backend_name=LVM-1
```

Kiểm tra các extra-specs

<img src="http://i.imgur.com/KezlTv4.png">

- Tạo một share dùng giao thức NFS kích thước 4GB , sử dụng backend lvm và có tên là `share-lvm-1`


```
manila create nfs 4 --name share-lvm-1 --share-type lvm
```
        - NFS là protocol
        - 4 là kích thước (GB)
        - `--name`: tên của share
        - `--share-type` chọn share-type

- Sau khi tạo share ta cần một đường dẫn để mount trên client

```
root@controller:/home/saphi# manila show share-lvm-1
+-----------------------------+-------------------------------------------------------------------------------------+
| Property                    | Value                                                                               |
+-----------------------------+-------------------------------------------------------------------------------------+
| status                      | available                                                                           |
| share_type_name             | lvm                                                                                 |
| description                 | None                                                                                |
| availability_zone           | nova                                                                                |
| share_network_id            | None                                                                                |
| export_locations            |                                                                                     |
|                             | path = 172.16.25.148:/var/lib/manila/mnt/share-ac9cd421-6f33-492f-8094-2dc21640bcb0 |
|                             | preferred = False                                                                   |
|                             | is_admin_only = False                                                               |
|                             | id = adc5a28e-fa64-464c-ac04-75aaa16911c6                                           |
|                             | share_instance_id = ac9cd421-6f33-492f-8094-2dc21640bcb0                            |
| share_server_id             | None                                                                                |
| host                        | storage@lvm1#lvm-single-pool                                                        |
| access_rules_status         | active                                                                              |
| snapshot_id                 | None                                                                                |
| is_public                   | False                                                                               |
| task_state                  | None                                                                                |
| snapshot_support            | True                                                                                |
| id                          | 7e478fbb-4867-4002-b9b5-34a56d585af0                                                |
| size                        | 4                                                                                   |
| name                        | share-lvm-1                                                                         |
| share_type                  | 79f0d08e-edc9-4409-9902-28cc69ad4d16                                                |
| has_replicas                | False                                                                               |
| replication_type            | None                                                                                |
| created_at                  | 2016-06-05T12:12:35.000000                                                          |
| share_proto                 | NFS                                                                                 |
| consistency_group_id        | None                                                                                |
| source_cgsnapshot_member_id | None                                                                                |
| project_id                  | 2dbc33e520b84b8ab550b649099d7972                                                    |
| metadata                    | {}                                                                                  |
+-----------------------------+-------------------------------------------------------------------------------------+
```

- Để client có thể mount được ta cần cấu hình access-allow. Với LVM sẽ là dựa trên địa chỉ IP, mặc định sẽ có quyền RW ta có thể thay đổi bằng thêm option `--access-level LEVEL` (RW, RO)


```
root@controller:/home/saphi# manila access-allow share-lvm-1 ip 172.16.25.153
+--------------+--------------------------------------+
| Property     | Value                                |
+--------------+--------------------------------------+
| share_id     | 7e478fbb-4867-4002-b9b5-34a56d585af0 |
| access_type  | ip                                   |
| access_to    | 172.16.25.153                        |
| access_level | rw                                   |
| state        | new                                  |
| id           | 272ff935-0b5c-4181-b802-48bf9c4e30d1 |
+--------------+--------------------------------------+
```
*Lưu ý: Các instance khi  mount phải được floating IPs vì LVM sẽ kiểm tra việc được truy cập hay không dựa trên IP này*

- Thực hiện mount trên 1 instance

<img src="http://i.imgur.com/GjsvLNS.png">

- Để delete share

```
manila delete test-share-1
```

#####Native GlusterFS Driver

- Yêu cầu phiên bản GlusterFS >= 3.6. Với glusterfs nếu cluster của bạn không hỗ trợ snapshot thì trên manila cũng sẽ mất đi tính năng này. Để cấu hình snapshot ta sẽ cấu hình Thin Provision theo bài hướng dẫn link

- Với bài lab của mình có 2 node và chạy kiểu replicate. Mình sẽ tạo các thinly provisioned và tạo volume trên đó.

Mô hình cài đặt

<img src="http://i.imgur.com/J4NlcLL.png">

Cài đặt glusterfs-v3.7

```
add-apt-repository ppa:gluster/glusterfs-3.7 -y
apt-get update
apt-get install glusterfs-server -y

```

Tham khảo script tạo thin LV và gluster volume

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

- Với backend là GlusterFS việc quản lý truy cập vào share file trên manila sẽ thông qua certificate do vậy ta cần cấu hình SSL cho client và server.

Tạo key và CA trên server. Mình sẽ copy key và ca tới tất cả server glusterfs
```
cd /etc/ssl
openssl genrsa -out glusterfs.key 1024
openssl req -new -x509 -key glusterfs.key -subj /CN=saphi -out glusterfs.pem
cp glusterfs.pem glusterfs.ca
```

Ở đây ta tạo CA có `CN=saphi` trên manila ta sẽ tạo access-allow cho CA của `saphi`



- Cấu hình GlusterFS tương tự như LVM mình sẽ sử dụng node compute để cài manila-share.




Cấu hình manila-share tại file `/etc/manila/manila.conf`

Enable backend glusterfsnative và protocol GLUSTERFS tại mục [DEFAULT]

```
enabled_share_backends= glusterfsnative #glusterfsnative là tên phần backend
enabled_share_protocols=NFS,CIFS,GLUSTERFS #giao thức cho phép

```
Cấu hình backend glusterfsnative như sau

```
[glusterfsnative]
share_backend_name = glusterfsnative # tên backend
glusterfs_servers = root@glusterfs1 # khai báo user,IP của gluster server, nếu có nhiều server glusterfs cách nhau bằng dấu phẩy
glusterfs_server_password = saphi # password để manila-host ssh vào glusterfs server
glusterfs_volume_pattern = manila-#{size}-.* # pattern volume trên manila khi tạo share mapping với volume trên glusterfs
share_driver = manila.share.drivers.glusterfs.glusterfs_native.GlusterfsNativeShareDriver # share driver sử dụng
driver_handles_share_servers = False  # False: không sử dụng driver xử lý share server
 
```

Nói thêm về volume-pattern:

Với script ở trên mình đã tạo các volume có mẫu `manila-$i-$j` với $i là size của volume và $j là số thứ tự. Do vậy ta sẽ cấu hình mẫu để các share manila khi tạo ra sẽ mapping với các volume trên glusterfs.

- Cài đặt glusterfs-client:

```
add-apt-repository ppa:gluster/glusterfs-3.7 -y
apt-get update
apt-get install glusterfs-client -y
```

- Sau khi cấu hình xong ta khởi động lại manila-share

```
service manila-share restart
```

- Kiểm tra service manila sẽ có thêm service cho glusterfsnative

<img src="http://i.imgur.com/wjUPlEq.png">

- Trên node `controller` cần phải enable protocol `GLUSTERFS` vì mặc định chỉ enable `NFS` và `CIFS`

thêm dòng sau vào mục [DEFAULT] trong `/etc/manila/manila.conf`

```
enabled_share_protocols=NFS,CIFS,GLUSTERFS
```
sau đó restart lại manila

```
service manila-api restart
service manila-scheduler restart
```

- Tạo share-type

```
manila type-create glusterfsnative False # tạo type tên là glusterfsnative DHSS = False
manila type-key glusterfsnative set share_backend_name=glusterfsnative # khai báo backend cho type này
```

<img src="http://i.imgur.com/qhm59tZ.png">

- Tạo share

```
manila create glusterfs 2 --name gluster-1 --share-type glusterfsnative
```
<img src="http://i.imgur.com/dGb2Fkb.png">

- Tạo access-allow cho CA có `CN=saphi`

<img src="http://i.imgur.com/fB1RSH7.png">

Ta thấy access-type ở đây là `cert`

- Kiểm tra đường dẫn mount path

<img src="http://i.imgur.com/CssIQzA.png">

- Thực hiện mount trên client

Client cần cài gói glusterfs-client

```
add-apt-repository ppa:gluster/glusterfs-3.7 -y
apt-get update
apt-get install glusterfs-client -y
```

Client cần  CA có `CN=saphi` và server phải biết CA này tôi sẽ copy key và CA trên server về client sau đó thực hiện mount

```
mount -t glusterfs glusterfs1:/manila-2-1 /mnt
```

<img src="http://i.imgur.com/lexNgpx.png">


#####GlusterFS Driver with NFS-Ganesha làm gateway

Sau khi cấu hình GlusterFS làm backend cho manila ta đã thấy một số nhược điểm

- Là dạng Volume mapping do vậy ta sẽ phải thực hiện tạo các volume trước khi tạo share.
- Nếu xem log các bạn có thể thấy để tạo một share thì manila-host sẽ phải SSH rất nhiều lần vào glusterfs-server(bằng số volume). Tham khảo log khi tạo share [manila-share.log](http://pastebin.com/vLMCPjmm)

Có một lựa chọn là ta có thể sử dụng NFS-Ganesha làm gateway cho GlusterFS Server sẽ có những ưu điểm sau

- Là dạng directory mapping, mỗi 1 share sẽ tạo ra 1 directory trên volume của glusterfs
- Dễ dàng giới hạn kích thước share (Không cần tạo nhiều volume có kích thước khác nhau trên glusterfs do vậy không cần cấu hình volume pattern)


Ta sẽ cần thêm 1 node cài nfs-ganesha server và mình sẽ sử dụng CentOS 7 để cài.
Mô hình cài đặt sẽ như sau

<img src="http://i.imgur.com/iiuSAJh.png">

######Cài đặt

- Cài đặt NFS-ganesha

download repo nfs-ganesha và glusterfs

```
curl -o /etc/yum.repos.d/nfs-ganesha.repo http://download.gluster.org/pub/gluster/glusterfs/nfs-ganesha/2.3.0/EPEL.repo/nfs-ganesha.repo
curl -o /etc/yum.repos.d/glusterfs-epel.repo http://download.gluster.org/pub/gluster/glusterfs/3.7/3.7.11/EPEL.repo/glusterfs-epel.repo
```
update repo

```
yum update
```

Cài đặt nfs-ganesha và glusterfs-client

```
yum -y install epel-release
yum -y install glusterfs-ganesha
```

Ta gluster server mình sẽ tạo một volume tên là `vol` để cấu hình trên ganesha
Ta có thể disable nfs trên volume này

```
gluster volume set vol nfs.disbale on
```

File cấu hình `/etc/ganesha/ganesha.conf` sẽ như sau

```
EXPORT
{
	# Export Id (bắt buộc, mỗi export có một đường Id)
	Export_Id = 78;

	# đường dẫn export (bắt buộc)
	Path = /shared_storage;

	# đường dẫn Pseudo  (yêu cầu cho NFS v4)
	Pseudo = /shared_storage;

	# Required for access (default is None)
	# Could use CLIENT blocks instead
	Access_Type = RW; # read/write

	# Exporting FSAL
	FSAL {
		Name = GLUSTER; # File system abstract layer là GLUSTER
		Hostname = "10.0.0.28"; # Ip, hostname của 1 node trong cụm glusterfs
		Volume =vol; # ten volume
	}
}

```

- Khởi động lại nfs-ganesha

```
service nfs-ganesha restart
```

- Kiểm tra export

```
showmount -e localhost
```

<img src="http://i.imgur.com/J2nwYAj.png">

- Cấu hình trên manila-share ở đây mình tiếp tục dùng node compute1 để chạy manila-share
mở file `/etc/manila/manila.conf` thêm đoạn sau

```
[ganesha]
share_backend_name = ganesha #tên backend
glusterfs_nfs_server_type = Ganesha # sử dụng ganesha server
glusterfs_target=root@10.0.0.28:/vol # user, IP(hostmame) , volume name của Gluster được cấu hình trên gluster
glusterfs_server_password=saphi # password để login vào gluster server
share_driver = manila.share.drivers.glusterfs.GlusterfsShareDriver # driver
ganesha_service_name = nfs-ganesha # tên service của ganesha trên ganesha-server
driver_handles_share_servers = False
glusterfs_ganesha_server_ip=172.16.25.146 # IP của ganesha server
glusterfs_ganesha_server_username=root # user để ssh
glusterfs_ganesha_server_password=saphi # password
```

sau đó enable backend này lên

```
enabled_share_backends = generic,ganesha
enabled_share_protocols = NFS,CIFS,GLUSTERFS
```
*Lưu ý:*

- Tại sao cần manila host cần ssh vào gluster? Bởi vì manila sẽ truy cập vào và đặt các quota cũng như các export trên volume đó. Ta có thể kiểm tra cmd_history trên glusterfs-server


<img src="http://i.imgur.com/JAfuv1N.png">




- Kiểm tra service manila

```
manila service-list
```

<img src="http://i.imgur.com/FImqfj7.png">

- Tạo share type ta cần disable snapshot vì đây là dạng directory mapping do vậy ko hỗ trợ snapshot

```

manila type-create ganesha False
manila type-key ganesha set share_backend_name=ganesha
manila type-key ganesha set snapshot_support=False

```
<img src="http://i.imgur.com/uLwJyl4.png">


- Tạo một share có tên `share-2` dung lượng 2GB trên backend `ganesha`

```
manila create nfs 2 --name share-2 --share-type ganesha
```
<img src="http://i.imgur.com/Rp88dFq.png">


- Set access allow theo IP

<img src="http://i.imgur.com/0wVD1nU.png">

Khi tạo một access ta thử kiểm tra file export share trên ganesha server tại đường dẫn `/etc/ganesha/export.d/`sẽ như sau

<img src="http://i.imgur.com/D90X63q.png">

- Kiểm tra đường dẫn mount

<img src="http://i.imgur.com/RoSg5Pq.png">

Ta thấy đường dẫn mount yêu cầu thêm `access-id` chính là ID của access ta tạo ở trên.

- Thực hiện mount kiểm tra trên client

<img src="http://i.imgur.com/1HBMUAj.png">

- Ta thấy mỗi một share được tạo ra sẽ là một export trên nfs-ganesha server và ứng với một share là một directory mới được tạo ra trên share server. Và các export được set đúng quota như share..
