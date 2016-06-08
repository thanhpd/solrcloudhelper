# CÀI ĐẶT SOLRCLOUD

![alt text](./Picture1.png "Apache Solr")

Bài viết này sẽ hướng dẫn cơ bản cách thiết lập SolrCloud trên hệ thống.

##### MỤC LỤC

<a name="requirement"/>
## YÊU CẦU
1. Đối với cụm cài đặt [ZooKeeper](https://zookeeper.apache.org/)

| Nền tảng  | Hỗ trợ môi trường Dev | Hỗ trợ môi trường Production |
|-----------|-----------------------|------------------------------|
| GNU/Linux | Có                    | Có                           |
| Solaris   | Có                    | Có                           |
| FreeBSD   | Có                    | Có                           |
| Win32     | Có                    | Không                        |
| Win64     | Có                    | Không                        |
| MacOS X   | Có                    | Không                        |

- Quorum: 1 (standalone mode) cho Môi trường Dev, 2N + 1 (cluster mode) cho môi trường Production
- Đảm bảo tài nguyên hệ thống như CPU, bộ nhớ, lưu trữ, mạng không bị quá giới hạn để hiệu suất của ZooKeeper không bị hạn chế
- Cần đồng bộ đồng hồ giữa các máy

*NÊN*
- Lưu transaction log của ZooKeeper trong 1 thiết bị riêng (device), lưu trên một phân vùng (partition) cũng có thể được tuy nhiên việc viết transaction log của ZooKeeper sẽ phải chia sẻ với các process khác
- Hạn chế khiến cho ZooKeeper thực hiện swap (GC thực hiện thường xuyên)

*NÊN TRÁNH*
- Sử dụng danh sách server ZooKeeper không nhất quán, ví dụ các máy chạy sử dụng danh sách ZooKeeper khác với danh sách đang có trong file config
- Lưu transaction log vào đĩa đã có hoạt động I/O lớn
- Đặt Java heap size sai, hoặc khiến swap 

2. Đối với cụm cài đặt [Solr](http://lucene.apache.org/solr/)
-tbd-

<a name="setup"/>
## CÀI ĐẶT
1. Chuẩn bị
- Cập nhật hệ thống: sudo yum update
- Cài đặt Java JDK 7: sudo yum install java-1.7.0-openjdk-devel
- Tạo biến môi trường JAVA_HOME dẫn đến Java JDK

2. Cài đặt ZooKeeper
- Download file cài đặt ZooKeeper: http://www-us.apache.org/dist/zookeeper/
(phiên bản ổn định mới nhất là 3.4.8)
- Giải nén file tới thư mục bất kì (vd: /usr/lib/zookeeper). *Nên đặt biến môi trường cho đường dẫn này, vd: ZK_HOME*
- Điều chỉnh file zoo_sample.cfg trong $ZK_HOME/conf, lưu thành file zoo.cfg **(bắt buộc)**; danh sách các parameter cần điều chỉnh được viết ở dưới cùng phần này

*Các parameter cơ bản trong ZooKeeper*
- *ticktime*: đơn vị thời gian trong ZooKeeper tính theo mili giây. Được dùng để tính heartbeat (check xem máy có đang hoạt động không) hoặc session timeout. *Không nên đặt thời gian này quá cao*
- *dataDir*: địa chỉ lưu snapshot, nên lưu ở thư mục riêng (vd: /etc/zookeeper/)
- *clientPort*: port mà client sẽ kết nối tới ZK

*Các parameter nâng cao trong ZooKeeper*
http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_advancedConfiguration
Lưu ý:
- dataLogDir: đường dẫn giữ log của ZooKeeper
- maxClientCnxns: số lượng kết nối tối đa từ client có thể chấp nhận được
- autopurge.snapRetainCount: số lượng snapshot & transaction log được giữ lại trong dataDir và dataLogDir; khi autopurge được chạy thì những phần snapshot & transaction log còn lại sẽ được xóa. (mặc định = 3; nhỏ nhất = 3)
- autopurge.purgeInterval: thời gian giữa những lần chạy purge. Mặc định = 0 (không chạy), đặt bằng số nguyên dương (> 1) để chạy.
- minSessionTimeout/maxSessionTimeout

*Các parameter cho cụm ZooKeeper*
http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_clusterOptions
Lưu ý:
- initLimit: số lượt tick mà giai đoạn sync có thể cần
- syncLimit: số lượt tick tối đa giữa một lần gửi request và nhận được acknowledgement
- server.x=[hostname]:nnnnn[:nnnnn]

