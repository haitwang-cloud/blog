> 本文是 [How to use ipset Command in Linux](https://www.thegeekdiary.com/how-to-use-ipset-command-in-linux/)的中文翻译版本，内容有删减


IP sets are stored collections of IP addresses, network ranges, MAC addresses, port numbers, and network interface names. The iptables tool can leverage IP sets for more efficient rule matching. For example, let’s say you want to drop traffic that originates from one of several IP address ranges that you know to be malicious. Instead of configuring rules for each range in iptables directly, you can create an IP set and then reference that set in an iptables rule. This makes your rule sets dynamic and therefore easier to configure; whenever you need to add or swap out network identifiers that are handled by the firewall, you simply change the IP set.

IP sets 是 IP 地址、网络范围、MAC 地址、端口号和网络接口名称的存储集合。iptables 工具可以利用 IP sets进行更高效的规则匹配。例如，假设您要丢弃来自已知为恶意的多个 IP 地址范围之一的流量。您可以创建一个 IP sets，然后在 iptables 规则中引用该 SET，而不是直接为 iptables 中的每个范围配置规则。这使您的规则集具有动态性，因此更易于配置;每当需要添加或交换由防火墙处理的网络标识符时，只需更改 IP sets即可。

The ipset command enables you to create and modify IP sets. First you need to set a name, storage method, and data type for your set, such as:

ipset 命令使您能够创建和修改 IP sets。首先，您需要为IP sets设置名称、存储方法和数据类型，例如：

```
# ipset create range_set hash:net
```

In this case, range_set is the name, hash is the storage method, and net is the data type. Then, you can add the ranges to the set:

**在上面的情况下，range_set是IP sets的名称，哈希是IP sets的存储方法，net 是数据类型。接下来，您可以将数据添加到IP sets中**：

```
# ipset add range_set 178.137.87.0/24
# ipset add range_set 46.148.22.0/24

```

Then, you use iptables to configure a rule to drop traffic whose source matches the ranges in this set:

然后，您可以使用 iptables 规则来丢弃source与set中的范围匹配的流量：

```
# iptables -I INPUT -m set --match-set range_set src -j DROP

```

Alternatively, to drop traffic whose destination matches the set:

或者，你可以设置丢弃target与集合set匹配的流量：

```
iptables -I OUTPUT -m set --match-set range_set dst -j DROP

```

## SYNTAX

The syntax of the ipset command is:

ipset 命令的语法是：

```
# ipset [options] {command}

```

### Blocking a list of network

Start by creating a new “set” of network addresses. This creates a new “hash” set of “net” network addresses named “myset”.

首先创建一个名为“myset”的hash 的set集。

```
# ipset create myset hash:net
```

or

```
ipset -N myset nethash
```

Add any IP address that you’d like to block to the set.

把您要阻止的IP 地址添加到上述的myset中。

```
# ipset add myset 14.144.0.0/12
# ipset add myset 27.8.0.0/13
# ipset add myset 58.16.0.0/15
# ipset add myset 1.1.1.0/24
```

Finally, configure iptables to block any address in that set. This command will add a rule to the top of the “INPUT” chain to “-m” match the set named “myset” from ipset (–match-set) when it’s a “src” packet and “DROP”, or block, it.

最后，配置 iptables 以阻止该set中的任何地址。 此命令将在“INPUT”链的顶部添加一条规则，以“-m”匹配来自 ipset (–match-set) 的名为“myset”的set，当它是来自“src”数据包时，“DROP” 它。

```
# iptables -I INPUT -m set --match-set myset src -j DROP

```

### Blocking a list of IP addresses

Start by creating a new “set” of ip addresses. This creates a new “hash” set of “ip” addresses named “myset-ip”.

首先创建一组新的 IP 地址的set。 这将创建一个名为“myset-ip”的新的“ip”地址hash set。

```
# ipset create myset-ip hash:ip
```

or

```
# ipset -N myset-ip iphash
```

Add any IP address that you’d like to block to the set.

把您要阻止的IP 地址添加到上述的myset中。
```
# ipset add myset-ip 1.1.1.1
# ipset add myset-ip 2.2.2.2
```

Finally, configure iptables to block any address in that set.

最后，配置iptables来阻止该set中的任何地址。

```
# iptables -I INPUT -m set --match-set myset-ip src -j DROP

```

### Making ipset persistent

The ipset you have created is stored in memory and will be gone after reboot. To make the ipset persistent you have to do the followings:

上面创建的 ipset 会存储在内存中，重启后将消失。 要使 ipset 持久化，您必须执行以下操作：


First save the ipset to **/etc/ipset.conf**:

首先将ipset保存到**/etc/ipset.conf**：
```
# ipset save > /etc/ipset.conf
```

Then enable ipset.service, which works similarly to iptables.service for restoring iptables rules.

然后启用 ipset.service，它的工作方式类似于 iptables.service，用于恢复 iptables 规则。

### Commands

To view the sets:

查看所有的sets
```
# ipset list
```

or

```
# ipset -L
```
2. To delete a set named “myset”:

删除一个名为“myset”的set

```
# ipset destroy myset
```

or
```
# ipset -X myset
```
To delete all sets:

删除所有的set

```
ipset destroy
```