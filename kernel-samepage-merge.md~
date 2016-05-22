#Kernel Samepage Merging in Virtualization

KSM (Kernel Samepage merging) là một tính năng của kernel cho phép hypervisor có thể chia sẻ các pages trong bộ nhớ giống hệt nhau cho các tiến trình khác nhau hoặc các VM khác nhau. 

KVM có thể sử dụng KSM để hợp nhất các pages trong memory của các VMs.
KSM thực hiện việc scan trong bộ nhớ chính -> Tìm kiếm các pages giống nhau. -> gộp các pages giống nhau thành một page. -> Page này được đánh dấu là "Copy On Write" có nghĩa là nếu có một tiến trình muốn modify page này. Nó sẽ tạo ra một page mới để thực hiện các thay đổi trên page đó.

Do vậy khi sử dụng KSM ta sẽ tốn một phần CPU để thực hiện việc Scan và Merge.
Kết quả : Tối ưu RAM nhưng CPU phải làm việc nhiều hơn.

Điều này sẽ hữu dụng cho các ứng dụng làm việc trên cùng một dữ liệu. Ví dụ các VMs trong KVM sử dụng chung 1 OS.

##Cấu hình KVM sử dụng KSM trên Ubuntu 14.04

KSM được tự động enable trong phiên bản kernel 2.6 trở đi. Để kiểm tra xem đã đã được enable chưa ta thực hiện lệnh sau

`cat /boot/config-`uname -r` | grep KSM`

nếu trả về `CONFIG_KSM=y` thì tức là KSM đã được enable.

Ta sẽ quan tâm các file trong thư mục `/sys/kernel/mm/ksm/`

<img src="http://i.imgur.com/nAwjLpo.png">

- full_scans: Thời gian(s)
- merge_across_nodes: Chỉ định nếu các pages từ các node khác nhau (?)
- pages_shared: Số pages đã shared
- pages_sharing: Số pages đang share
- pages_to_scan: Số pages sẽ scan
- pages_unshared: Số pages khác nhau nhưng sẽ liên tục kiẻm tra các pages này cho việc merge
- pages_volatile: bao nhiêu pages thay đổi quá nhanh để đặt trong 1 cây (?)
- run: 
	- 0: disable scan
	- 1: enable scan
	- 2: disable scan and unmerging
- sleep_millisecs: tổng thời gian ksmd sleep (ms)

###Test launch VM trên KVM

Thực hiện launch 10 VM Cirros cấu hình RAM 512MB trên KVM

Kiểm tra Ram sau khi Launch 10 VM

<img src="http://i.imgur.com/OFt08eP.png">

Thực hiện enable KSM

`echo "1" > /sys/kernel/mm/ksm/run`

Cho phép scan 20000 pages 

`echo "20000" > /sys/kernel/mm/ksm/pages_to_scan`

Kiểm tra RAM đang sử dụng

<img src="http://i.imgur.com/S6ju6Fm.png">

Ta thấy lượng RAM giảm đi là khoảng 600MB. Các chỉ số về `pages_shared` và `pages_sharing` như hình.

Kết luận: Ta có thể sử dụng KSM với mục đích tạo ra nhiều VM nhưng không quan tâm hiệu năng của các VM (sẽ bị giảm đi)
Các chỉ số `page_shared` và `pages_sharing` nếu càng cao càng tốt. Nếu chỉ số `pages_unshared` rất cao thì xem xét có nên sử dụng KSM không. 




