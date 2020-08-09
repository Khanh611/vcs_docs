#  LINUX NETWORK NAMESPACES

# 1. Cài đặt Open Vswitch

Trên distro Ubuntu, để cài đặt Open Vswitch một cách đơn giản nhất, bạn có thể sử dụng `apt-get` hoặc `aptitude` để cài đặt từ các gói .deb:

- Các gói deb `openvswitch-switch` và `openvswitch-common` là các gói để cài đặt các thành phần lõi userspace của openvswitch. Nếu không thực hiện nghiên cứu sâu thì chỉ với 2 gói trên là đã đủ cho bạn vọc vạch về OVS.

	```
	sudo apt-get install openvswitch-switch openvswitch-common -y
	```

- Với kernel datapath, `openvswitch-datapath-dkms` có thể được cài đặt để tự động xây dựng và cài đặt các module Open Vswitch kernel cho kernel đang chạy của bạn.

- Với DPDK datapath, Open vSwitch với hỗ trợ DPDK được gói trong gói `openvswitch-switch-dpdk`.

Hoặc để cài đặt đầy đủ các thành phần mới nhất của Open Vswitch, tham khảo hướng dẫn cài đặt từ source code tại: http://docs.openvswitch.org/en/latest/intro/install/general/ 

# 2. Một số bài lab thử nghiệm tính năng của linux network namespaces

## 2.1. Kết nối 2 máy ảo thuộc 2 VLAN sử dụng Openvswitch

### 2.1.1 Kết nối thông qua virtual ethernet (veth)

*topo*

### - Tạo 2 namespace

- Để tạo mới một namespace:

```
ip netns add ns1
ip netns add ns2
```

2 namespace được tạo mới: **ns1** & **ns2**

Liệt kê các network-namespace:

```
ip netns list
```

- Thực thi lệnh (commands) trong namespaces

Ví dụ chạy lệnh `ip a` trong namespace có tên **ns1**

``` sudo ip netns exec ns1 ip a ```

### - Thêm Virsual switch nsbr

`ovs-vsctl add-br nsbr`

- Kiểm tra OVS bridge

`ovs-vsctl show`

*pic show*

#### - Thêm cặp veth

- Để kết nối các namespace tới swtich, sử dụng veth pairs. 

- ***Virtual Ethernet interfaces (hay veth)*** là một kiến trúc thú vị, chúng luôn có 1 cặp, và được sử dụng để kết nối như một đường ống: các lưu lượng tới từ một đầu veth và được đưa ra, peer tới giao diện veth còn lại. Như vậy, có thể dùng veth để kết nối mạng trong namespace từ trong ra ngoài root namespace trên các interface vật lý của root namespace.

- Thêm một veth pairs sử dụng lệnh: 

	`ip link add veth0 type veth peer name veth1`

	Khi đó, một veth được tạo ra với 2 đầu là 2 interface veth0 và veth1. 

- Như mô hình trong bài, thêm veth nối giữa namespace **ns1** và switch **nsbr** :

	`ip link add veth-ns1 type veth peer name eth0-ns1`

- Thêm veth dùng để nối giữa **ns2** và **nsbr**: 

	`ip link add veth-ns2 type veth peer name eth0-ns2`
  
  *pic ip link*
  
  #### - Gán các interface vào namespace tương ứng

- Chuyển interface eth0-ns1 vào namespace ns1 và eth0-ns2 vào namespace ns2 và bật lên:

	```
	ip link set eth0-ns1 netns ns1
  ip netns exec ns1 ip link set lo up
	ip netns exec ns1 ip link set eth0-ns1 up

	ip link set eth0-ns2 netns ns2
  ip netns exec ns2 ip link set lo up
	ip netns exec ns2 ip link set eth0-ns2 up
	```

- Các interface còn lại gán vào openvswitch port và set VLAN: 

	```
	ip link set veth-ns1 up
	ip link set veth-ns2 up
	ovs-vsctl add-port ovs1 veth-ns1 -- set port veth-ns1 tag=10
	ovs-vsctl add-port ovs1 veth-ns2 -- set port veth-ns2 tag=20
	```

	- ***Lưu ý***: khi thêm 2 đầu interface của các veth vào namespace hoặc openvswitch thì phải bật các device lên (device hiểu đơn giản trong trường hợp này là 2 đầu interface của veth). Sử dụng câu lệnh: `ip link set <device_name> up`

- Kiểm tra lại sử dụng câu lệnh: `ovs-vsctl show` được kết quả như sau: 

*pic check*

### - Gán địa chỉ IP 

```
ip netns exec ns1 ip addr add 10.0.0.1/24 dev eth0-ns1
ip netns exec ns2 ip addr add 10.0.0.2/24 dev eth0-ns2
```

