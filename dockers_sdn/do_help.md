```shell
do_help()
{
    welcome

    DEBUG "
说明:
	################################$TIME###################################
	计算资源Docker 化， kubelet 化，自动化
        Create By: zhangjie
        Email: zhangjie@letv.com
	################################$TIME###################################
    "

    DEBUG "
    命令:
    ${CMD} check

    ${CMD} etcd <cluster token> <etcd_ca_bin_file> <local addr> <member1_addr> <member2_addr>
    ${CMD} docker <dns1,dns2,dns3,..., you can no input it>
    ${CMD} minion <interface 例如：eth2  或者：eth2,eth3 > <k8s master endpoint> <localAddr> <etcd_client_ca.tar path such /root/sftp/etcd_client_ca.tgz> <etcd1ip> <etcd2ip> <etcd3ip> <dns1,dns2,dns3,... you can no input it>
    ${CMD} master <k8s master endpoint> <etcd_client_ca.tar path such /root/sftp/etcd_client_ca.tar> <etcd1ip> <etcd2ip> <etcd3ip> <dns1,dns2,dns3,... you can no input it>

    ${CMD} cnictl <etcd1ip> <etcd2ip> <etcd3ip>
    ${CMD} cnitest <container ip> <ip prefix 子网掩码(如 22 24)> <gateway> <tag>

    ${CMD} registry <s3_accesskey> <s3_secretkey> <s3_endpoint> <s3_bucket> <auth_server_endpoint> <notification_endpoint> <notification_token>
    ${CMD} auth-server <core_endpoint>

    ${CMD} rootimage <username> <passwd>

    ${CMD} build-agent <core_endpoint> <authorization_config_token> <local_addr> <build_root_path> <maven_cache_rootpath> <etcd_ca_client_ca_gzip> <build_process_num>

    ${CMD} sd <core_endpoint> <authorizationConfigToken> <localAddr> <k8s_etcd_endpoints> <k8s_endpoint> <k8s_etcd_ca_client_gzip>

    ${CMD} events <relay_address or relay domain name , must start at http://> <kubernetes_address>

    ${CMD} core <etcd_ca_client_gzip> <etcd_endpoints> <mysql_endpoint> <mysql_username> <mysql_password> <mysql_database> <slb_endpoint> <login_token> <authorization_config_token> <registry_endpoint> <registry_username> <registry_password> <wuzei_endpoint> <wuzei_poolname> <wuzei_key>  <notification_token> <influxdb_address> <influxdb_username> <influxdb_password>

    ${CMD} heapster <relay_address or relay domain name , must start at http://> <kubernetes_address>
    ${CMD} influxdb
    ${CMD} influxdb-relay <local_address> <influxdb1> <influxdb2>
    ${CMD} kapacitor <local_address> <influxdb_name.rp, default: k8s.default> <relay_address or relay domain name , must start at http://>
    ${CMD} alert <local_address> <leengine_core_address> <leengine_core_token> <influxdb_address> <influxdb_dbname> <influxdb_username> <influxdb_password>


    ${CMD} upgrade [可选参数: cni cnictl registry pause core gbalancer build-agent sd events heapster influxdb influxdb-relay kapacitor alert]
    参数:
    interface: 代表的是网卡名字，将来这个网卡需要挂载到 ${BR0} ovs 网桥上, 强烈建议这个网卡是万兆网卡！！！ 此网卡必须没有配置IP地址, 没有配置
    localAddr: 一定是本机的IP 地址, 千万不要写错
    etcd_client_ca.tar: 链接etcd 的client 公钥， 需要提前传到master 服务器上,是一个文件的绝对路径
    cluster token: etcd 集群同步使用的token，强烈建议相同集群下的token是一致的，但是不同集群下的token最好不一致
    leengineca_bin_file: leengine ca 的私钥安装文件， 必须是一个绝对路径

    日志文件: ${LOG_FILE}
    "

    WARN "强烈建议在执行 minion 命令前重启一下当前主机 ..."
}
```
