# MRCP
[![Discord](https://img.shields.io/discord/937821572285206659?style=flat-square&label=Discord&logo=discord&logoColor=white&color=7289DA)](https://discord.com/servers/teknology-hub-937821572285206659)

## Tổng quan

Steam Manifest Request Code Provider (MRCP) là một bot nhẹ cung cấp mã yêu cầu manifest Steam qua một socket UDP sử dụng tài khoản Steam của bạn.

## Cách sử dụng

Tải xuống tệp nhị phân cho hệ điều hành của bạn tại releases hoặc tự biên dịch (điều này liên quan đến Linux vì Native AOT thêm các phụ thuộc động vào các phiên bản thư viện cụ thể), chạy nó một lần với tham số --setup để nhập thông tin xác thực tương tác, sau đó bạn có thể chạy nó mà không cần bất kỳ tham số nào (ví dụ như một dịch vụ), nó sẽ lắng nghe trên cổng đã chỉ định cho các yêu cầu UDP.

## Định dạng dữ liệu yêu cầu và phản hồi

Yêu cầu (16 byte):
+ uint AppId
+ uint DepotId
+ ulong ManifestId

Phản hồi (8 byte): ulong ManifestRequestCode

## Tại sao bạn cần nó

Trong khi TEK Steam Client có thể cài đặt các ứng dụng Steam tùy ý, nó không thể lấy mã yêu cầu manifest cho các ứng dụng không được sở hữu trên tài khoản mà nó đang đăng nhập, và do đó không thể tải xuống những ứng dụng đó. MRCP cho phép proxy các yêu cầu MCR đến một tài khoản sở hữu các ứng dụng liên quan mà không làm rò rỉ thông tin xác thực tài khoản vì nó chạy trên một máy chủ từ xa.

## Client side code example

```cs
using System.Collections.Frozen;
using System.Net.Sockets;
using TEKSteamClient;

static readonly IPEndPoint ServerEndpoint = IPEndPoint.Parse("*Your server IP*:*MRCP Port*");
static ulong GetManifestRequestCode(uint appId, uint depotId, ulong manifestId)
{
	using var socket = new Socket(AddressFamily.InterNetwork, SocketType.Dgram, ProtocolType.Udp); //May make it a static singleton
	Span<byte> buffer = stackalloc byte[16];
	BitConverter.TryWriteBytes(buffer[..4], appId);
	BitConverter.TryWriteBytes(buffer[4..8], depotId);
	BitConverter.TryWriteBytes(buffer[8..], manifestId);
	socket.SendTo(buffer, ServerEndpoint);
	socket.Receive(buffer);
	return BitConverter.ToUInt64(buffer);
}

//The following code needs to be executed only once per app lifetime, you may put it in the beginning of Main method
CMClient.ManifestRequestCodeSourceOverrides = System.Collections.Frozen.FrozenDictionary.ToFrozenDictionary((IEnumerable<KeyValuePair<uint, Func<uint, uint, ulong, ulong>>>)
	[
		new(/*A depot ID*/, GetManifestRequestCode),
		new(/*Another depot ID*/, GetManifestRequestCode)
	]);

//Your code using TEK Steam Client goes here, it'll automatically send requests to MRCP when needed
```

## License

MRCP is licensed under the [MIT](https://github.com/Nuclearistt/MRCP/blob/main/LICENSE) license.
