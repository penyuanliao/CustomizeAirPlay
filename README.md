CustomizeAirPlay
================
mDNS(Multicast DNS)原理
mDNS所使用的Port使用 5353 Port(IETF組織訂定)。

Apple Bonjour協定制定Multicast IPAaddress 224.0.0.251 

作用: mDNS就是來解決在區域網路內如何取得雙方的動態IP。

在Apple 的設備上（電腦，筆記本，iphone，ipad等設備）都提供了這個服務。很多Linux設備也提供這個服務。Windows的設備可能沒有提供，但是如果安裝了iTunes之類的軟件的話，也提供了這個服務。
jmDNS是一個JAVA平台的，提供mDNS服務的第三方庫。在這個jar包引入到Android項目裡，就可以獲得mDNS服務了。

協定: http://www.ietf.org/rfc/rfc1035.txt

封包結構: http://m.oschina.net/blog/290922

DNS: http://www.csie.dyu.edu.tw/~swang/I2N/CH11.pdf


Service Type : link (_ipp._tcp)


DatagramSocket

```ActionScript
//送出UDP Socket
var udpSocket:DatagramSocket = new DatagramSocket();
udpSocket.addEventListener(DatagramSocketDataEvent.DATA, dataHandler);
udpSocket.addEventListener(IOErrorEvent.IO_ERROR, errorHandler);
	
/**
 * 	Send standard DNS requests.
 *  http://www.networksorcery.com/enp/rfc/rfc1035.txt
 */
private function sendQuery():void
{
	// Don't bind until first message sent out and only bind once
	if (udpSocket.localAddress == null)
	{
		udpSocket.bind(7000);
		udpSocket.receive();			
		
	}				
	trace("Receiving on: " + udpSocket.localAddress+":"+udpSocket.localPort + "\n");
	var bytes:ByteArray = new ByteArray();
	
	// DNS Header
	bytes.writeFloat(0); 
	bytes.writeByte(0); // First 5 bytes, Transaction ID: 0x0000, Query, Opcode: 0x000 etc..
	bytes.writeByte(1);	// 1 Question
	bytes.writeByte(0); // No Answer RRs
	bytes.writeByte(0); // No Authority RRs
	bytes.writeFloat(0); // No Additional RRs, and 3 bytes of 0s
	// Query Name Chunk
	writeQueryName(bytes, service.text);
	// Write out Type and Class (PTR/12) and 00 01
	bytes.writeByte(0); // 00
	bytes.writeByte(12); // 0C
	bytes.writeByte(0); // 00
	bytes.writeByte(1); // 01
	
	bytes.writeShort(0x21); //Type
	bytes.writeByte(0x80); // FlushCache
	bytes.writeByte(0x01); // Class IN
	bytes.writeUnsignedInt(0); //Time to live
	bytes.writeShort(8 + mDNSAddress.length); //Data length
	bytes.writeShort(0x00); //priority
	bytes.writeShort(0x00); //weight
	bytes.writeShort(7000); //port
	writeString(bytes, mDNSAddress);
	bytes.writeByte(0x00); //end
	//TXT Record
	bytes.writeShort(0xC00C); //Name ^^^
	bytes.writeShort(0x10); //Type TXT
	bytes.writeByte(0x80); //Flush Cache
	bytes.writeByte(0x01); // Class IN
	bytes.writeUnsignedInt(120);//TTL
	bytes.writeShort(String(txt.text).length); //Data length
	writeString(bytes, txt.text); //Data
	
	// Send DNS packet to mDNS port and address, since we are sending from a 
	// non-5353 port it will send the response back as a unicast DNS answer
	udpSocket.send(bytes, 0, 0, mDNSAddress, mDNSPort);
}
/**
 * 	Splits the service mDNS string and packs the bytes 
 *  correctly in the <code>bytes</codes> parameter.
 */
private function writeQueryName(bytes:ByteArray, query:String = "_home-sharing._tcp"):void
{
	var arr:Array = query.split(".");
	var len:int = arr.length;
	for (var i:int = 0; i < len; i++)
	{
		bytes.writeByte((arr[i] as String).length);
		bytes.writeUTFBytes(arr[i]);
	}
	// Add .local and 00
	bytes.writeByte(5);
	bytes.writeUTFBytes("local");
	bytes.writeByte(0);
}
private function writeString(bytes:ByteArray, string:String):void
{
	var ar:Array = string.split(".");
	var len:int = ar.length;
	
	for (var i:int = 0; i < len; i++)
	{
		bytes.writeByte((ar[i] as String).length);
		bytes.writeUTFBytes(ar[i]);
	}
}
	
```
