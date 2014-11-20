CustomizeAirPlay
================
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
