CustomizeAirPlay
================
```ActionScript
////////////////////////////////////////////////////////////////////////////////
// 
//  Copyright (c) 2010 Renaun Erickson <renaun.com>
// 
//  Permission is hereby granted, free of charge, to any person
//  obtaining a copy of this software and associated documentation
//  files (the "Software"), to deal in the Software without
//  restriction, including without limitation the rights to use,
//  copy, modify, merge, publish, distribute, sublicense, and/or sell
//  copies of the Software, and to permit persons to whom the
//  Software is furnished to do so, subject to the following
//  conditions:
// 
//  The above copyright notice and this permission notice shall be
//  included in all copies or substantial portions of the Software.
// 
//  THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
//  EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES
//  OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND
//  NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT
//  HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY,
//  WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING
//  FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR
//  OTHER DEALINGS IN THE SOFTWARE.
// 
////////////////////////////////////////////////////////////////////////////////
/**
 Flex SDK Coding Conventions Used:
 http://opensource.adobe.com/wiki/display/flexsdk/Coding+Conventions
 **/
package net
{
	import flash.utils.ByteArray;
	
	import mx.utils.ObjectUtil;

	/**
	 * 	http://www.networksorcery.com/enp/rfc/rfc1035.txt
	 * 
	 * 	4. MESSAGES
	 *  4.1. Format
	 */
	public class DNSMessage
	{
		public function DNSMessage()
		{
		}
		
		//--------------------------------------------------------------------------
		//
		//  Variables
		//
		//--------------------------------------------------------------------------
		
		//----------------------------------
		//  subsection
		//----------------------------------
		
		//--------------------------------------------------------------------------
		//
		//  Properties
		//
		//--------------------------------------------------------------------------
		
		public var transactionID:int = 0;
		
		public var questions:Vector.<DNSResourceRecord> = new Vector.<DNSResourceRecord>();

		public var answers:Vector.<DNSResourceRecord> = new Vector.<DNSResourceRecord>();

		public var authorities:Vector.<DNSResourceRecord> = new Vector.<DNSResourceRecord>();

		public var additionals:Vector.<DNSResourceRecord> = new Vector.<DNSResourceRecord>();
		
		public var allResourceRecords:Vector.<DNSResourceRecord> = new Vector.<DNSResourceRecord>();
		
		//--------------------------------------------------------------------------
		//
		//  Methods
		//
		//--------------------------------------------------------------------------
		
		public function parse(packet:ByteArray):void 
		{
			questions.length = 0;
			answers.length = 0;
			authorities.length = 0;
			additionals.length = 0;
			allResourceRecords.length = 0;
			
			// read of Transaction ID
			transactionID = packet.readUnsignedShort();
			// Reading off QR, Opcode, AA, TC, RD, RA, Z, and RCode for now
			// TODO: check for error code
			packet.readUnsignedShort();
			// Counts
			var totalCount:int = 0;
			var answerStart:int = packet.readUnsignedShort();
			var authorityStart:int = packet.readUnsignedShort() + answerStart;
			var additionalStart:int = packet.readUnsignedShort() + authorityStart;
			totalCount = packet.readUnsignedShort() + additionalStart;
			
			var resourceBucket:Vector.<DNSResourceRecord> = questions;
			for (var i:int = 0; i < totalCount; i++)
			{
				if (i >= additionalStart)
					resourceBucket = additionals
				else if (i >= authorityStart)
					resourceBucket = authorities;
				else if (i >= answerStart)
					resourceBucket = answers;

				var resource:DNSResourceRecord = new DNSResourceRecord();
				parseResourceRecord(packet, resource, (resourceBucket == questions));
				resourceBucket.push(resource);
				allResourceRecords.push(resource);
			}
		}
		
		/**
		 * 
		 */
		private function parseResourceRecord(packet:ByteArray, resource:DNSResourceRecord, isQuestion:Boolean):void
		{
			// Name
			resource.name = readDNSString(packet);
			
			var type:int = packet.readUnsignedShort();
			var classType:int = packet.readUnsignedShort();
			resource.classType = classType; // Most always 1 = IN
			if (isQuestion)
				return;
			resource.ttl = packet.readUnsignedInt();
			var dataLength:int = packet.readUnsignedShort();
			
			switch (type)
			{
				case 1:
					resource.type = DNSResourceRecord.TYPE_A;
					resource.data = packet.readUnsignedByte() + "." + packet.readUnsignedByte() + "." + packet.readUnsignedByte() + "." + packet.readUnsignedByte();
					break;
				case 5:
					resource.type = DNSResourceRecord.TYPE_CNAME;
					// TODO might be wrong
					resource.data = packet.readUTFBytes(dataLength);
					break;
				case 12:
					resource.type = DNSResourceRecord.TYPE_PTR;
					resource.data = readDNSString(packet);
					break;
				case 33:
					resource.type = DNSResourceRecord.TYPE_SRV;
					resource.data = new Object();
					resource.data.priority = packet.readUnsignedShort();
					resource.data.weight = packet.readUnsignedShort();
					resource.data.port = packet.readUnsignedShort();
					resource.data.target = readDNSString(packet);
					break;
				case 16:
					resource.type = DNSResourceRecord.TYPE_TXT;
					resource.data = new Object();
					var len2:int = 0;
					var pos:int = 0;
					var parts:Array = [];
					while (pos < dataLength)
					{
						len2 = packet.readUnsignedByte();
						parts = packet.readUTFBytes(len2).split("=");
						resource.data[parts[0]] = parts[1];
						pos += len2 + 1;
					}
					break;
				default:
					resource.type = DNSResourceRecord.TYPE_UNSUPPORTED;
					resource.data = new ByteArray();
					packet.readBytes(resource.data, 0, dataLength);
					break;				
			}
		}
		
		/**
		 * 	Handles reading DNS strings and watches out for compression
		 *  pointers.
		 */
		private function readDNSString(packet:ByteArray):String
		{
			var value:String = "";
			var len:int = packet.readUnsignedByte();
			var lastPosition:int = 0;
			var offset:int = 0;
			while (len > 0)
			{
				if (len > 63)
				{
					// TODO Only reading byte which limits offset to 256, rfc shows it could be more
					offset = packet.readUnsignedByte(); 
					if (lastPosition < packet.position)
						lastPosition = packet.position;
					packet.position = offset;
					len = packet.readUnsignedByte();
				}
				value += packet.readUTFBytes(len);
				len = packet.readUnsignedByte();
				if (len > 0)
					value += ".";
			}
			// If pointer reset offset to last position
			if (offset > 0)
			{
				packet.position = lastPosition;
				offset = 0;
				lastPosition = 0;
			}
			return value;
		}
	}
}
```
