# network-scripts package for RHEL9-based distros
The actual "NetworkManager" seems to be almost perfect.
But still remain a number of corner cases when the old good "network-scripts" is the best solution.

While the "NetworkManager" is the best for desktop, "network-scripts" is the good choice for server-side.
They does nothing unexpected at runtime. But can generate rather complex network topology.
E.g., (eth - bond - bridge - vlan - bridge - macvlan).
Or something like (eth - bridge - veth - bridge - [openvswitch-managed area]).

I just got "initscripts-10.00.18-1.el8.src.rpm" and rebuilt it to fit the actual dependencies.

There is also the "network-scripts-extra" package that enhances "network-scripts" functions.
It helps to tune extra devices, as well as do some Ethernet tuning.
Feel free to use "network-scripts-extra" as a template to write Your own enhancements.
(M.b., for "NetworkManager". It supports hooks :)

The example on how to replace "NetworkManager" (touch "sysconfig/disable-deprecation-warnings" to stop warnings).
It assumes there is the only physical interface, but You can fix it by tuning "dev=".
 
    rpm -ivh network-scripts-*

    systemctl enable network.service
    #yum -y remove NetworkManager
    systemctl disable NetworkManager

    dev=`ls -1 /sys/class/net/ | grep -v '^lo$' | sort | head -n 1`
    IPADDR=$(ip -4 a show dev ${dev} | awk '{if ($1 == "inet") print $2}' | cut -d '/' -f 1)
    PREFIX=$(ip -4 a show dev ${dev} | awk '{if ($1 == "inet") print $2}' | cut -d '/' -f 2)
    GATEWAY=$(ip r | awk '{if ($1 == "default") print $3}')
    HWADDR=$(ip l show dev ${dev} | awk '{if ($1 == "link/ether") print $2}')

    M1="11111111111111111111111111111111" # 32 bit mask
    M0="00000000000000000000000000000000" # 32 bit mask
    mask=${M1::$PREFIX}${M0:$PREFIX:32} # Leading "1", trailing "0"
    BYTE_MASK=$((2#${mask:0:8})).$((2#${mask:8:8})).$((2#${mask:16:8})).$((2#${mask:24:8})) # 4 octets

    cd /etc/sysconfig/network-scripts/
    cat <<EOF> ifcfg-${dev}
    #
    DEVICE="${dev}"
    ONBOOT="yes"
    HWADDR="${HWADDR}"
    TYPE="Ethernet"
    BOOTPROTO="static"
    IPADDR="${IPADDR}"
    NETMASK="${BYTE_MASK}"
    GATEWAY="${GATEWAY}"
    EOF

Good luck!