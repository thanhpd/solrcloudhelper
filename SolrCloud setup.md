# CÀI ĐẶT SOLRCLOUD

![alt text](./Picture1.png "Apache Solr")

Bài viết này sẽ hướng dẫn cơ bản cách thiết lập SolrCloud trên hệ thống.

##### MỤC LỤC
[Yêu cầu](#requirement)  
[Cài đặt](#setup)  

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
- Đặt Java heap size sai, hoặc khiến swap.  


2. Đối với cụm cài đặt [Solr](http://lucene.apache.org/solr/)
-tbd-

<a name="setup"/>
## CÀI ĐẶT  

### Chuẩn bị  
- Cập nhật hệ thống: `$ sudo yum update`
- Cài đặt Java JDK 7: `$ sudo yum install java-1.7.0-openjdk-devel`
- Tạo biến môi trường `JAVA_HOME` dẫn đến Java JDK

### Cài đặt ZooKeeper
1. Download file cài đặt ZooKeeper: http://www-us.apache.org/dist/zookeeper/
(phiên bản ổn định mới nhất là 3.4.8)
- Giải nén file tới thư mục bất kì (vd: /usr/lib/zookeeper). *Nên đặt biến môi trường cho đường dẫn này, vd: ZK_HOME*
- Điều chỉnh file zoo_sample.cfg trong `$ZK_HOME/conf`, lưu thành file `zoo.cfg` **(bắt buộc)**; danh sách các parameter có thể điều chỉnh được viết ở dưới cùng phần này 

**Ví dụ:**

Config cho môi trường Dev
```xml
tickTime=2000
dataDir=/var/lib/zookeeper/
clientPort=2181
```
Config cho môi trường Production
```xml
tickTime=2000
dataDir=/var/lib/zookeeper/
clientPort=2181
initLimit=5
syncLimit=2
server.1=zoo1:2888:3888
server.2=zoo2:2888:3888
server.3=zoo3:2888:3888
# zoo1, zoo2, zoo3 được định nghĩa trong host file của từng máy, có thể thay 3 địa chỉ này bằng IP trực tiếp
```
- Trong trường hợp thiết lập cụm ZK, thêm file `myid` với 1 kí tự duy nhất là số thứ tự của server hiện tại vào thư mục `$ZK_HOME\data` (Xem thêm ở phần Các parameter cho cụm ZooKeeper bên dưới)
- Lặp lại thiết lập cho các máy khác
- Danh sách các command cơ bản trong ZooKeeper có ở dưới

***Các command cơ bản trong ZooKeeper***
- Khởi động ZooKeeper:  
`$ sudo bash $ZK_HOME\bin\zkServer.sh start`        
- Kiểm tra trạng thái ZooKeeper:  
`$ sudo bash $ZK_HOME\bin\zkServer.sh status`  
Mode: standalone - chạy chế độ 1 node  
Mode: leader - chế độ cluster, máy đang chạy command là leader  
Mode: follower - chế độ cluster, máy đang chạy command là follower  
Nếu có lỗi thì check ở thư mục của biến `logDir` trong ZooKeeper  
- Dừng ZooKeeper:  
`$ sudo bash $ZK_HOME\bin\zkServer.sh stop`
- Khởi động lại ZooKeeper:  
`$ sudo bash $ZK_HOME\bin\zkServer.sh restart`
- Danh sách command đầy đủ:  
`$ sudo bash $ZK_HOME\bin\zkServer.sh help`

***Kết nối tới cụm ZooKeeper (có thể hiểu như truy cập directory của ZooKeeper)***  
`$ sudo bash $ZK_HOME\bin\zkCli.sh -server [host1:port1,host2:port2,host3:port3,...]`  
Sau khi kết nối gõ `help` để có danh sách câu lệnh

***Các parameter cơ bản trong ZooKeeper***
- `ticktime`: đơn vị thời gian trong ZooKeeper tính theo mili giây. Được dùng để tính heartbeat (check xem máy có đang hoạt động không) hoặc session timeout. *Không nên đặt thời gian này quá cao*
- `dataDir`: địa chỉ lưu snapshot, nên lưu ở thư mục riêng (vd: /etc/zookeeper/)
- `clientPort`: port mà client sẽ kết nối tới ZK

***Các parameter nâng cao trong ZooKeeper***  
http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_advancedConfiguration  
Lưu ý:
- `dataLogDir`: đường dẫn giữ log của ZooKeeper
- `maxClientCnxns`: số lượng kết nối tối đa từ client có thể chấp nhận được
- `autopurge.snapRetainCount`: số lượng snapshot & transaction log được giữ lại trong dataDir và dataLogDir; khi autopurge được chạy thì những phần snapshot & transaction log còn lại sẽ được xóa. (mặc định = 3; nhỏ nhất = 3)
- `autopurge.purgeInterval`: thời gian giữa những lần chạy purge. Mặc định = 0 (không chạy), đặt bằng số nguyên dương (> 1) để chạy.
- `minSessionTimeout`/`maxSessionTimeout`

***Các parameter cho cụm ZooKeeper***  
http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_clusterOptions  
Lưu ý:
- `initLimit`: số lượt tick mà giai đoạn sync có thể cần
- `syncLimit`: số lượt tick tối đa giữa một lần gửi request và nhận được acknowledgement
- `server.x=[hostname]:nnnnn[:mmmm]`
Địa chỉ các server trong ZK ensemble, trong đó:
`x` là số thứ tự của server, được định nghĩa trong file myid của thư mục `$ZK_HOME/data`;
`hostname` có thể là địa chỉ trong mạng LAN hoặc public IP, cần đảm bảo cho các máy cùng ZK ensemble và Solr có thể kết nối được với nhau;
Có 2 port: `nnnn` được dùng để kết nối tới máy leader trong cụm ZK; `mmmm` được dùng cho bầu cử leader (leader election);

### Cài đặt Solr
https://cwiki.apache.org/confluence/display/solr/Taking+Solr+to+Production

- Tải bộ cài đặt Solr tại: http://www.apache.org/dyn/closer.lua/lucene/solr  
Phiên bản ổn định nhất có thể dùng 5.5.1 hoặc 6.0.1
- Cài đặt lsof cho CentOS để script cài đặt có thể kiểm tra hoạt động Solr: `$ sudo yum install lsof`
- Giải nén và chạy script cài đặt bằng cách thực hiện câu lệnh:
```bash
$ sudo tar xzf solr-{version}.tgz solr-{version}/bin/install_solr_service.sh --strip-components=2
$ sudo bash ./install_solr_service.sh solr-{version}.tgz
```
Mặc định script sẽ giải nén Solr binary ở thư mục `/opt/solr` và những file được Solr viết sẽ ở thư mục `/var/solr`. Script sẽ tạo user Solr và gán quyền cho 2 thư mục kể trên.
Script mặc định khi chạy sẽ tương đương với câu lệnh:
```bash
$ sudo bash ./install_solr_service.sh solr-X.Y.Z.tgz -i /opt -d /var/solr -u solr -s solr -p 8983
```
Danh sách các lựa chọn của script cài đặt có thể lấy bằng cách chạy câu lệnh:
```bash
$ sudo bash ./install_solr_service.sh -help
```
- Sau khi cài đặt xong, ta có thể kiểm tra bằng cách chạy câu lệnh:
```bash
$ sudo service solr status
```
- Tạo chroot (znode) cho cụm ZooKeeper bằng cách chạy script:
```bash
$ZK_INSTALL_DIR server/scripts/cloud-scripts/zkcli.sh -zkhost host1:port1,host2:port2,host3:port3 -cmd makepath /solr
```
- Chỉnh sửa file `/etc/default/solr.in.sh` để override các thuộc tính cho SolrCloud. Ví dụ
```xml
# Specify ZK_HOST to connect to ZooKeeper and enables SolrCloud mode
ZK_HOST=host1:port1,host2:port2,host3:port3/solr
# Specify SOLR_HOST to identify the current Solr node with ZooKeeper cluster
SOLR_HOST=solr1.example.com
```
- Kiểm tra trạng thái của Solr