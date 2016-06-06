##Cấu hình Openstack Nova sử dụng Hugepages

Bài lab thực hiện trên phiên bản Openstack Mitaka và OS là Ubuntu 14.04

## Cấu hình hugepages cho node compute1

- Kiểm tra hugepagessize `cat /proc/meminfo | grep Huge`
- Node compute của tôi 6GB RAM nên tôi sẽ sử dụng 4GB cho các instance sử dụng  Hugepages.

- Để lưu giữ cấu hình hugepages ta cần thêm tham số vào khi khởi động ubuntu.
tôi thêm dòng sau vào file `/etc/default/grub`

```
GRUB_CMDLINE_LINUX="$GRUB_CMDLINE_LINUX hugepagesz=2M hugepages=2000 transparent_hugepage=never"

```

- Update grub `update-grub`


- Mount phân vùng `hugepages` khi khởi động tôi thêm dòng sau vào file `/etc/fstab`

```
mkdir /hugepages

echo "hugetlbfs    /hugepages    hugetlbfs    defaults    0 0" >> /etc/fstab

```

- Mặc định libvirt sẽ không có quyền mount sang phân vùng này do bị Appparmor chặn do vậy ta sẽ thực hiện bỏ chặn bằng lệnh sau

```
touch /etc/apparmor.d/disable/usr.sbin.libvirtd

```

- Cấu hình libvirt sử dụng HUGEPAGES ta sửa `KVM_HUGEPAGES=1` trong file `/etc/default/qemu-kvm`

- Reboot node compute


##Cấu hình trên node controller
- Tạo aggregate khai báo node compute nào được cấu hình để sử dụng hugepages

```
nova aggregate-create hugepages-aggr
nova aggregate-set-metadata hugepages-aggr hpgs=true
nova aggregate-add-host hugepages-aggr compute1
```

<img src="http://i.imgur.com/RhuG5TY.png">

- Để khi launch instance thì nova cần biết instance nào sẽ sử dụng hugepage ta sẽ tạo một flavor mới và thêm `extra_specs` vào flavor đó

```
nova flavor-create m1.tiny.hugepages auto 512 2 1
nova flavor-key m1.tiny.hugepages set hw:mem_page_size=2048
nova flavor-key m1.tiny.hugepages set aggregate_instance_extra_specs:hpgs=true

```

<img src="http://i.imgur.com/uCsline.png">

- Thêm giá trị `AggregateInstanceExtraSpecsFilter` vào thông số `scheduler_default_filters` để nova-scheduler fil host theo `extra_specs` trong file `/etc/nova/nova.conf`

```
scheduler_default_filters=RetryFilter,AvailabilityZoneFilter,RamFilter,DiskFilter,ComputeFilter,ComputeCapabilitiesFilter,ImagePropertiesFilter,ServerGroupAntiAffinityFilter,ServerGroupAffinityFilter,AggregateInstanceExtraSpecsFilter

```

- Khởi động lại nova-scheduler
```
service nova-scheduler restart

```

- Boot 1 instance mới

```
openstack server create --image cirros --flavor m1.tiny.hugepages --nic net-id=net-admin cirros-hugepages-demo

```

- Kiểm tra hugepage free lúc này sẽ bị giảm 256 pages

<img src="http://i.imgur.com/idfY8WS.png">


- Kiểm tra file XML của instance `cirros-hugepages-demo`

<img src="http://i.imgur.com/LaJ30a4.png">
