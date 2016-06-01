### Tìm hiểu hypercontainer

####Tổng quan Hypercontainer

Hypercontainer là một "Hypervisor agnostic Docker runtime" mà cho phép bạn chạy các image của Docker trên bất kỳ hypervisor nào (KVM, Xen, ... )

Về mặt kỹ thuật có thể nói

```
Hypercontianer = Hypervisor + Kernel + Docker image
```

Bằng việc bao gồm các ứng dụng ên trong các VM các nhau và các kernel spaces, Hypercontainer có thể thực thi việc cô lập phần cứng (Hardware-enforced Isolation) rất tốt, mà cần nhiều trong môi trường multi-tenant.

Hypercontainer cũng hứa hẹn việc Infrastructure không thay đổi (Immutable Infrastructure) bằng việc loại bỏ middle layer của Guest OS, cùng với những rắc rối để cấu hình và quản lý chúng

<img src="https://trello-attachments.s3.amazonaws.com/55545e127c7cbe0ec5b82f2b/879x320/5471e40d4a519c3d31f455bdccc978ca/upload_2_3_2016_at_3_50_31_PM.png">

Về mặt hiệu năng, Hypercontainer là siêu nhẹ

- Chưa tới một giây để Boot: mili giây để launch một Hypercontainer
- Slimmed Footprint: khoảng 28MB RAM

Với Hypercontainer, tương lai của container as a Service chỉ xung quanh góc 

<img src="https://trello-attachments.s3.amazonaws.com/552ba9ad83b51945d06ef23b/940x238/9e7346bfd21bc756361c70d8397e76f2/upload_2015-04-13_at_7.58.15_pm.png">



####Hypercontainer hoạt động như thế nào

Hyper có 4 thành phần: 
- CLI: hyperctl
- Daemon: hyperd ( REST APIs)
- Guest kernel: hyperkernel
- Guest init service: hyperstart

Trên một host linux vật lý thực hiện như sau

```
#hyperctl pull nginx:latest
#hyperctl run nginx:latest

```

Sau khi chạy, Hyper sẽ launch các docker image với 1 VM instance hay vì các container

```
#docker ps
#hyperctl list

```

Bên trong HyperVM, Một linux kernel tối giản gọi là `HyperKernel` đươc boot. Kernel này sử dụng một service init nhỏ được gọi là `Hyperstart` để load các image Docker từ host, cài đặt MNT namespace dể cô lập filesystem của chúng và thực hiện launch chúng.

<img src="https://trello-attachments.s3.amazonaws.com/554c998a4c9dacc5c143ec99/1083x635/c8748abc93dbc18e70f7a09d2963e8ff/hyper.png">




