# AS 相关

> 以下信息通过 https://bgp.he.net/ 查得

| Origin AS (ASN 数据) | Announcement (CIDR | Description |
| -- | -- | -- |
| AS55960 | 54.222.0.0/19 | Beijing Guanghuan Xinwang Digital Technology co.Ltd. |
| AS4808 | 124.65.0.0/16 <br> 124.65.192.0/18 | China Unicom Beijing province network |
| AS4808 | 202.96.13.0/24 | China National Instruments Import & Export Corp |
| AS4837 | 219.158.96.0/19 | CNC group |
| AS4134 | 202.97.0.0/19 | CHINANET backbone network |
| AS9394 | 61.236.0.0/15 <br> 61.237.0.0/16	 <br> 61.237.0.0/17 | China TieTong Telecommunications Corporation |
| AS9808 | 218.200.0.0/13 <br> 218.204.0.0/14 <br> 218.206.0.0/15 <br> 218.207.192.0/19 <br><br> 112.0.0.0/10 <br> 112.5.0.0/16 <br><br> 211.136.0.0/13 <br> 211.140.0.0/14	<br> 211.142.0.0/15 | China Mobile Communications Corporation |
| AS9808 | 211.143.144.0/20 | China Mobile Communications Corporation - fujian |
| | xx通 | |
| AS9929 | | China Netcom Backbone |
| AS9800 | | CHINA UNICOM |
| AS4538 | | China Education and Research Network Center |
| AS9306 | | China International Electronic Commerce Center |
| AS4799 | | CNCGROUP Jitong IP network |
| | 城域网 | |
| AS17623 | | China Unicom Shenzen network |
| AS17816 | | China Unicom IP network China169 Guangdong province |
| | IDC | |
| AS4816 | | China Telecom (Group) |
| AS23724 | | IDC, China Telecommunications Corporation |
| AS4835 | | China Telecom (Group) |
| | ISP | |
| AS9812 | | Oriental Cable Network Co., Ltd. |
| | 中国电信北方九省 | |
| AS17785 | | asn for Henan Provincial Net of CT |
| AS17896 | | asn for Jilin Provincial Net of CT |
| AS17923 | | asn for Neimenggu Provincial Net of CT |
| AS17897 | | asn for Heilongjiang Provincial Net of CT |
| AS17883 | | asn for Shanxi Provincial Net of CT |
| AS17799 | | asn for Liaoning Provincial Net of CT |
| AS17672 | | asn for Hebei Provincial Net of CT |
| AS17638 | | ASN for TIANJIN Provincial Net of CT |
| AS17633 | | ASN for Shandong Provincial Net of CT |
| | IXP/NAP | |
| AS4847 | | China Networks Inter-Exchange |
| AS4839 | | NAP2 at CERNET located in Shanghai |
| AS4840 | | NAP3 at CERNET located in Guangzhou |
| | CN2 | |
| AS4809 | | China Telecom Next Generation Carrier Network |
| | 教育城域网 | |
| AS9806 | | Beijing Educational Information Network Service Center Co., Ltd |
| | IPv6 Test Network | |
| AS23912 | | China Japan Joint IPv6 Test Network |
| AS9808 | | Guangdong Mobile Communication Co.Ltd. |

## Origin AS

> ref: https://www.arin.net/resources/originas.html

The **Origin Autonomous System** (`AS`) field is an optional field collected by `ARIN` during all IPv4 and IPv6 block transactions (allocation and assignment requests, reallocation and reassignment actions, transfer and experimental requests). This additional field is used by IP address block holders (including legacy address holders) to record a list of the **Autonomous System Numbers** (`ASNs`), separated by commas or whitespace, from which the addresses in the address block(s) may originate.

Collecting and making Origin AS information available to the community is part of the implementation of Policy [ARIN-2006-3: Capturing Originations in Templates](https://www.arin.net/vault/policy/proposals/2006_3.html), included in the ARIN Number Resource Policy Manual (`NRPM`) Section 3.5: "[Autonomous System Originations](https://www.arin.net/policy/nrpm.html#three5)." This information is available using our [Bulk Whois](https://www.arin.net/resources/request/bulkwhois.html) service.

## AS 相关信息

- http://www.caida.org/home/ -- 一个研究 AS 级的拓扑结构的网站，在这个网站可以找到因特网 AS 级的拓扑资料和各种分析；其分析数据的三个来源是：
    - http://www.routeviews.org/ - BGP
    - http://www.caida.org/tools/measurement/skitter/ -- RouterTrace
    - Whois
- http://www.caida.org/analysis/topology/as_core_network/historical.xml -- 全球 AS 爆炸性增长的一个直观印象
- [自治系统 - 维基百科](https://zh.wikipedia.org/wiki/%E8%87%AA%E6%B2%BB%E7%B3%BB%E7%BB%9F)
- [Exploring Autonomous System Numbers - The Internet Protocol Journal - Volume 9, Number 1 - Cisco](http://www.cisco.com/c/en/us/about/press/internet-protocol-journal/back-issues/table-contents-12/autonomous-system-numbers.html)
- [Request Resources](https://www.arin.net/resources/request/asn.html)

Resources

- 最完整： [Autonomous System (AS) Numbers](http://www.iana.org/assignments/as-numbers/as-numbers.xml)
- 最客观： [Mapping Local Internet Control - Country Report: China](http://cyber.law.harvard.edu/netmaps/country_detail.php/?cc=CN)
- 最全面： [AS info for the country China](http://www.tcpiputils.com/browse/as/cn)
- 最专业： [Networks of China - bgp.he.net](http://bgp.he.net/country/CN/)
- 最清晰： [Whois](http://ipwhois.cnnic.cn/)

参考：

- http://www.cnblogs.com/webmedia/archive/2006/01/28/324031.html
- https://github.com/idealhack/notes/blob/master/notes/BGP.md



