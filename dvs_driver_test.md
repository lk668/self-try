1、使用jenkin构建vio的时候配置文件选择vio-dvs.json

2、登录到构建好的vio的loadbalancer节点创建虚机（需要创建key），在compute节点和controller节点上创建会连不到创建的虚机上，网络选择默认的
flat-tempest，在computenode1上创建之后在computenode2同样创建一台（使用availability-zone），创建之后使用ssh登录到虚机上（使用key）
ping对方虚机以确定当前网络模式下虚机是否互通

3、登录到controller01节点上。切换成ml2 core plugin，neutron server启动失败，看log里面有fernet相关信息，报参数不够。neutron ml2启动
的时候会建立到nova的连接，需要进程启动用户的key，用来解析配置文件中的password，但是vio之前没有这个需求（因为不是ml2 core plugin）。这里
可以执行下面操作，从现有key中拷贝一个给root用户，因为是用root用户启动的进程。 
        ```
        cd /etc/fernet
        cp nova.key root.key
        ```

4、拷贝文件https://github.com/huihongxiao/vmware-nsx/blob/master/vmware_nsx/plugins/ml2/vds_mech_driver.py
至vmware_nsx/plugins下。

5、修改neutron.conf配置文件，将core plugin改为ml2, 之后重启neutron-server服务

6、为vds mech driver添加entry point
/usr/lib/python2.7/dist-packages/vmware_nsx-10.0.0.6420508.egg-info/entry_points.txt
        ```
        [neutron.ml2.mechanism_drivers]
        vds = vmware_nsx.plugins.vds_mech_driver:VDSMechDriver
        ```

7、为了能创建vxlan network，在tenant_network_types添加vxlan；将vds加入到 neutron conf下的[ml2]的mechanism_drivers
中；然后配置vlan的network_vlan_ranges.综上，在neutron conf后面添加：
        ```
        [ml2]
        tenant_network_types = vxlan, vlan
        mechanism_drivers = vds

        [ml2_type_vxlan]
        vni_ranges = 1:1000

        [ml2_type_vlan]
        network_vlan_ranges = physical_net:1:1000
        ```
        
 8、回到loadbalancer01，创建网络neutron net-create, 之后在该网络中创建子网neutron subnet-create，之后使用新的网络
 分别在两个cluster上创建虚机，失败，提示端口绑定错误，另vds_mech_driver.py在进程启动时没有创建pyc文件。后在singleVM环
 境中重做实验，可以创建虚机，但是ping不通，依旧是网络问题，neutron日志报vds_mech_driver.py代码问题。目前等待代码更新。。。
