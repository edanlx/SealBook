协议:TCP/IP是通信协议(网络协议)的总称。互联网中具有代表性的协议有IP、TCP、HTTP等。而LAN(局域网)中常用的协议IPX/SPX等。TCP/IP家族主要包含IP、ICMP、TCP、UDP、HTTP、TELNET、SNMP、SMTP等。协议主要由ISO制定国际标准OSI
分组交换协议:将大数据分隔成包(package),每一组数据都包含报文首部用于标记属于原始数据的哪一部分
OSI7层参考模型:1.物理层、2.数据链路层、3.网络层、4.传输层(TCO/UDP等)、5.会话层、6.表示层、7.应用层(HTTP\FTP等)。层与层之间用接口进行连接，每一层可以换成其它的协议而其它层不受影响
【参考模型图】

【通信与七个分层】
应用层工作：处理/展示用户所能看到的东西。表示层工作:以UTF-8进行编码/解码。会话层工作:何时发送、何时建立连接，建立几个连接进行管理，并不具备实际传输功能。传输层工作:建立连接与断开连接重发。网络层工作:与传输层协作实现可靠传输,避免出现丢失数据，顺序错误等问题。数据链路层、物理层:将数据的0、1转化为电压和脉冲传输给物理介质。

传输方式的分类:
1.面向有连接型,连接打开才可传输数据，以TCP为代表。例如打电话，必须存在才可拨号
2.面向无连接型,无需确认连接是否打开，可以直接发送，UDP为代表。例如邮寄，有地址即可邮寄，即使其并不存在