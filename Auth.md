## OpenVPN auth bằng Username/Password + otp + tls-auth.

### Cài đặt Mysql-server
```
apt-get install mysql-server -y
```
#### Tạo user và database chứa thông tin login cho OpenVPN
```
mysql
CREATE DATABASE openvpn;
GRANT ALL ON openvpn.* TO 'openvpn'@"%" IDENTIFIED BY '123456';
USE openvpn;
```
Create table user
```
CREATE TABLE IF NOT EXISTS `user` (
    `user_id` varchar(32) COLLATE utf8_unicode_ci NOT NULL,
    `user_pass` varchar(32) COLLATE utf8_unicode_ci NOT NULL DEFAULT '1234',
    `user_mail` varchar(64) COLLATE utf8_unicode_ci DEFAULT NULL,
    `user_phone` varchar(16) COLLATE utf8_unicode_ci DEFAULT NULL,
    `user_secret` varchar(64) COLLATE utf8_unicode_ci DEFAULT NULL,
    `user_online` tinyint(1) NOT NULL DEFAULT '0',
    `user_enable` tinyint(1) NOT NULL DEFAULT '1',
    `user_start_date` date NOT NULL,
    `user_end_date` date NOT NULL,
PRIMARY KEY (`user_id`),
KEY `user_pass` (`user_pass`)
) ENGINE=MyISAM DEFAULT CHARSET=utf8 COLLATE=utf8_unicode_ci;
```
Create table log
```
CREATE TABLE IF NOT EXISTS `log` (
    `log_id` int(10) unsigned NOT NULL AUTO_INCREMENT,
    `user_id` varchar(32) COLLATE utf8_unicode_ci NOT NULL,
    `log_trusted_ip` varchar(32) COLLATE utf8_unicode_ci DEFAULT NULL,
    `log_trusted_port` varchar(16) COLLATE utf8_unicode_ci DEFAULT NULL,
    `log_remote_ip` varchar(32) COLLATE utf8_unicode_ci DEFAULT NULL,
    `log_remote_port` varchar(16) COLLATE utf8_unicode_ci DEFAULT NULL,
    `log_start_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `log_end_time` timestamp NOT NULL DEFAULT '0000-00-00 00:00:00',
    `log_received` float NOT NULL DEFAULT '0',
    `log_send` float NOT NULL DEFAULT '0',
PRIMARY KEY (`log_id`),
KEY `user_id` (`user_id`)
) ENGINE=MyISAM  DEFAULT CHARSET=utf8 COLLATE=utf8_unicode_ci;
```

Tạo user data
Script tạo user
[Create user](gen-otp.py)

### Cài đặt OpenVPN và EasyRSA

#### Cài đặt OpenVPN
```
sudo apt install openvpn
```

#### Cài đặt EasyRSA
#### Cài đặt easyrsa từ git
```
cd && wget https://github.com/OpenVPN/easy-rsa/releases/download/v3.0.5/EasyRSA-nix-3.0.5.tgz 
```
#### Copy easy-rsa vào thư mục /etc/openvpn/
```
- cp -r EasyRSA-3.0.5 /etc/openvpn/
```

#### Tạo file vars chứa thông tin cài đặt Easy-RSA
```
- cd /etc/openvpn/EasyRSA-3.0.5/
- vim vars
```

######## Copy vào file vars theo thông tin dưới đây:
```
set_var EASYRSA                 "$PWD"
set_var EASYRSA_PKI             "$EASYRSA/pki"
set_var EASYRSA_DN              "cn_only"
set_var EASYRSA_REQ_COUNTRY     "VN"
set_var EASYRSA_REQ_PROVINCE    "HaNoi"
set_var EASYRSA_REQ_CITY        "HaNoi"
set_var EASYRSA_REQ_ORG         "vccloud-labs CERTIFICATE AUTHORITY"
set_var EASYRSA_REQ_EMAIL       "openvpn@vccloud.vn"
set_var EASYRSA_REQ_OU          "vccloud-labs EASY CA"
set_var EASYRSA_KEY_SIZE        2048
set_var EASYRSA_ALGO            rsa
set_var EASYRSA_CA_EXPIRE       7500
set_var EASYRSA_CERT_EXPIRE     365
set_var EASYRSA_NS_SUPPORT      "no"
set_var EASYRSA_NS_COMMENT      "vccloud-labs CERTIFICATE AUTHORITY"
set_var EASYRSA_EXT_DIR         "$EASYRSA/x509-types"
set_var EASYRSA_SSL_CONF        "$EASYRSA/openssl-1.0.cnf"
set_var EASYRSA_DIGEST          "sha256"
```

Sau đó lưu và thoát. Ấn Esc sau đó :wq!.

#### Thêm quyền cho file vars được phép thực thi.
```
- chmod +x vars
```
## Tạo keys cho OpenVPN
Trong bước này chúng ta sẽ tạo key cho OpenVPN dựa trên easy-rsa và file 'vars' đã được tạo.
Chúng ta sẽ tạo CA keys, Server và Client keys, DH và CRL PEM file
Chúng ta sẽ build tất cả các keys trên sử dụng câu lệnh easyrsa:
```
- cd /etc/openvpn/EasyRSA-3.0.5/
```
#### Khởi tạo và xây dựng CA
Trước khi tạo mọi loại keys, chúng ta cần khởi tạo thư mục PKI và xây dựng CA.
```
- ./easyrsa init-pki
- ./easyrsa build-ca
```
Nhập password cho CA key và chúng ta có thể lấy ca.crt và ca.key trong thư mục PKI
#### Tạo Server key
Tạo server với tên ví dụ 'vccloud-server' sử dụng câu lệnh dưới đây
```
- ./easyrsa gen-req vccloud-server nopass
```
nopass = tùy chọn vô hiệu hóa mật khẩu cho vccloud-server
Tiếp theo assign vccloud-server sử dụng CA certificate.
```
- ./easyrsa sign-req server vccloud-server
```
Sau đó Enter và chúng ta có thể lấy vccloud-server.crt trong thư mục pki/issued/
Kiểm tra certificate file sử dụng câu lệnh OpenSSL và để đảm bảo rằng nó không bị lỗi

#### Tạo Client key
Để tạo key cho client. Chúng sẽ tạo key mới với tên 'client01'
Để tạo client01 sử dụng câu lệnh bên dưới:
```
./easyrsa gen-req client01 nopass
```
Bây giờ để sign client01 sử dụng CA certificate như cách bên dưới:
```
- ./easyrsa sign-req client client01
```
Sau đó xác nhận bằng 'yes'. Sau đó kiểm tra lại sử dụng OpenSSL command:
```
- openssl verify -CAfile pki/ca.crt pki/issued/client01.crt
```
#### Tạo Diffie-Hellman Key
Việc này sẽ làm tốn một chút thời gian tùy thuộc vào độ dài có sẵn trên server.
Để tạo Diffie-Hellman key sử dụng câu lệnh sau:
```
- ./easyrsa gen-dh
```
DH key sẽ được tạo trong thư mục pki

#### Tùy chọn khóa CRL
CRL (Certificate Revoking List) được sử dung trong việc thu hồi client keys.
Nếu bạn có nhiều client key và muốn thu hồi bớt sử dụng câu lệnh sau:

```
./easyrsa revoke clientkey
```
clientkey = tên của một client key nào đó ví dụ 'client01'

Để tạo CRL key sử dụng câu lệnh sau
```
./easyrsa gen-crl
```

#### Tạo TLS key ta.key
```
cd /etc/openvpn/
openvpn --genkey --secret ta.key
cp ta.key /etc/openvpn/server
cp ta.key /etc/openvpn/client
```
CRL PEM sau khi tạo sẽ được để trong PKI

#### Copy Certificate files
###### Copy server key và certificate

```
cp pki/ca.crt /etc/openvpn/server/
cp pki/issued/vccloud-server.crt /etc/openvpn/server/
cp pki/private/vccloud-server.key /etc/openvpn/server/
```

###### Copy client01 key và certificate

```
cp pki/ca.crt /etc/openvpn/client/
cp pki/issued/client01.crt /etc/openvpn/client/
cp pki/private/client01.key /etc/openvpn/client/
```

###### Copy DH và CRL Key.

```
cp pki/dh.pem /etc/openvpn/server/
cp pki/crl.pem /etc/openvpn/server/
```
##### Create script auth

Tạo thư mục chứ script
```
mkdir /etc/openvpn/script
cd /etc/openvpn/script
```

Tạo script auth-pass-otp
[Script authenticaton](script/auth-user-pass-otp.py)

Tạo script connect khi user login thành công update trong database column online
[Script connect](script/connect.sh)

Tạo script disconnect khi user disconnect sẽ update trong database column online
[Script disconnect](script/disconnect.sh)

Tạo script firewall để mỗi team chỉ được truy cập đến LAN được cho phép.
Script control access team DEV
[Script dev-up.sh](script/dev-up.sh)
[Script dev-down.sh](script/dev-down.sh)

Script control access team Sysadmin
[Script ops-up.sh](script/ops-up.sh)
[Script ops-down.sh](script/ops-down.sh)

Script control access team Admicro
[Script bigdata-up.sh](script/bigdata-up.sh)
[Script bigdata-down.sh](script/bigdata-down.sh)

##### File config OpenVPN

```
vim /etc/openvpn/ops.conf
vim /etc/openvpn/dev.conf
vim /etc/openvpn/bigdata.conf
```
Nội dung của file config của team Sysadmin [ops.conf](ops.conf)
Nội dung của file config của team DEV [dev.conf](dev.conf)
Nội dung của file config của team Admicro [bigdata.conf](bigdata.conf)

###### Thay đổi quyền trong thư mục OpenVPN
```
chmod -R 755 /etc/openvpn
```
###### Khởi động dịch vụ OpenVPN server
```
service openvpn@ops restart
service openvpn@dev restart
service openvpn@bigdata restart
```
###### Tạo config file client
```
cd /etc/openvpn/client
vim client.ovpn
```
Với cấu hình sau

[Client config file](client.ovpn)

