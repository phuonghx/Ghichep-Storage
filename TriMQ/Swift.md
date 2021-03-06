#Kiến trúc và mô hình dữ liệu của Swift

#Mục lục
**Table of Content**

- [1. Mô hình dữ liệu Swift](#1)
	- [1.1 Account](#11)
	- [1.2 Container](#12)
	- [1.3 Object](#13)
- [2. Kiến Trúc Swift](#2)
	- [2.1 Regions](#21)
	- [2.2 Zone](#22)
	- [2.3 Node](#23)
	- [2.4 Chính sách lưu trữ](#24)
- [3. Tiến trình xử lý của server](#3)
- [4. Tiến trình thống nhất](#4)
- [5. Định vị dữ liệu](#5)
	- [5.1 Cơ bản Ring: Hàm băm](#51)
	- [5.2 Cơ bản Rings: Hàm băm phù hợp.](#52)
	- [5.3 Rings: Sửa đổi băm phù hợp](#53)
- [6. Phân phối dữ liệu](#6)
- [7. Tạo và cập nhật Rings](#7)

-----------------------------------------------------------------
	
<a name="1"></a>
##1. Mô hình dữ liệu Swift

<img src="http://i.imgur.com/qePcTmO.png">

OpenStack Swift cho phép người dùng lưu trữ đối tượng dữ liệu phi cấu trúc với tên chuẩn bao gồm: tài khoản(account), Containner, và đối tượng. Sử dụng 1 hoặc nhiều thông tin cho phép hệ thống tạo ra 1 vị trí lưu trữ duy nhất cho dữ liệu.

/account: Tài khoản vị trí lưu trữ là 1 tên khu vực lưu trữ duy nhất, nơi sẽ chứa các metadata(thông tin mô tả) về tài khoản này, cũng như nơi chứa tài khoản. Lưu ý trong Swift 1 tài khoản ko phải là 1 danh tính người dùng mà là 1 khu vực lưu trữ.

/account/container: là vùng lưu trữ được định nghĩa phía trong account nơi mà các metadata về container và danh sách các object mà container đó lưu trữ.

/account/container/object: vị trí lưu trữ đối tượng là nơi dữ liệu đối tượng và metadata của nó đc lưu trữ.

3 thông số này liên kết với nhau để tạo nên các địa điểm lưu trữ là duy nhất, vì vậy container và object không cần là duy nhất trên cả cluster.

Tiếp theo là địa điểm lưu trữ (storage location) và cách tạo nên địa điểm lưu trữ đó(có 3 thông tin đại diện: account, container, object):


<a name="11"></a>
###1.1 Accounts: 

Account này là vị trí lưu trữ gốc(root) cho dữ liệu. Các account này có tên duy nhất trên toàn bộ hệ thống. Mỗi account này đều có cơ sử dữ liệu lưu trữ metadata cho account và danh sách các container trong account đó. Header account metadata được lưu trữ như 1 cặp key-value(tên-value) với các đảm bảo độ bền của tất cả dữ liệu Swift.
Đáng chú ý là database của account không chưa đối tượng hoặc metadata của đối tượng. Account database được truy cập khi người sử dụng (người đc cấp phép hoặc dịch vụ đc ủy quyền) yêu cầu về metadata của account hoặc danh sách của các container trong account.
Tài khoản được thiết kế để người dùng đơn lẻ hoặc multiuser truy cập tùy thuộc vào quyền được thiết lập. Với các quyền được thiết lập đúng, Swift user có thể tạo, thay đổi loại bỏ các container và các đối trượng trong account đó.
Phân biệt giữa Swift account(storage) và Swift user(identity).

<a name="12"></a>
###1.2 Container:
Được định nghĩa là 1 phần của không gian tên tròg acoount cung cấp vị trí lưu trữ trữ, nơi mà các đối tượng sẽ được tìm thấy. Container không thể lồng vào nhau, khái niệm này tương tự như khái niệm thư mục trong filesystem, nó cũng cung cấp vị trí lưu trữ (/account/container/object). Việc đặt tên container là không hạn chế trên toàn cầu, 2 container tên giống nhau ở 2 account khác nhau là hoàn toàn hợp lệ, số lượng container trong cùng 1 account là không giới hạn.
Giống như mỗi tài khoản có 1 cơ sở dữ liệu thì container cũng có cơ sở dữ liệu cho nó. Mỗi cơ sở dữ liệu này đều chưa metadata cho cotainer và 1 bản ghi cho mỗi object mà nó chứa. Header metadata container cũng được lưu trữ như 1 cặp khóa-giá trị đảm bảo độ bền cho dữ liệu.Database của container sẽ không chứa các đối tượng mà nó sẽ đc truy cập khi người dùng yêu cầu danh sách các đối tượng trong container hoặc metadata của container.
Trong account bạn có thể tạo và sử dụng container để nhóm các dữ liệu theo cách của bạn. Bạn cũng có thể dùng metadata với các công cụ, tính năng của Swift  để sử dụng cho các conng việc khác.

<a name="13"></a>
###1.3 Object: 
Là dữ liệu được lưu trữ trong Swift. nó có thể là ảnh, video, document, log file, backup, snapshot....Mỗi đối tượng bao gồm metadata của nó và dữ liệu của chính nó.
Metadata header được lưu trữ như 1 cặp khóa-value(tên-giá trị) với tất đảm bảo độ bền của đối tượng dữ liệu và không yêu cầu thêm độ trễ. Metadata có thể cung cấp thông tin quan trọng về đối tượng. Ví dụ Khi lưu trữ 1 video chúng ta có thể lưu trữ về nội dung, định dạng, thời gian, 1 tài liệu có thể chưa thông tin tác giả, 1 trình tự sẽ chứa quá trình mà nó đc tạo ra.
Mỗi object phảu thuộc về 1 container. Khi 1 đối tượng đc lưu trữ tên 1 cumk, người dùng sẽ luôn luôn tham chiếu đến vị trí lưu trữ đối tượng (account/container/object). Không có giới hạn về số lượng mà các object có thể lưu trữ trong 1 container
Từ quan điểm của người dùng, vị trí lưu trữ đối tượng là nơi đối tượng đc tìm thấy. Như đã nói thì Swift lưu trữ nhiều bản sao dữ liệu trên các ổ cứng khác nhau để đảm bảo độ tin cậy và sẵn sàng của dữ liệu. Swift thực hiện điều này bằng các đặt các object trong 1 nhóm hợp lý gọi là partion(phân vùng) Mỗi Partion sẽ map tới nhiều ổ đĩa, mỗi bản copies sẽ được lưu trong 1 ổ cứng. Người dùng sẽ không biết và không cần biết nơi lưu trữ thực sự của các đối tượng


<a name="2"></a>
##2. Kiến Trúc Swift

<img src="http://i.imgur.com/tQKSUvf.png">
Dữ liệu Swift (account, container, object) là những tài nguyên cuối cùng được lưu trữ trên ổ cứng vật lý. 1 máy chạy Swift process sẽ là 1 node, 1 cluster sẽ là nhóm các node chạy tập trung đầy đủ quy trình và các dịch vụ cần thiết để hoạt động như 1 hệ thống lưu trữ phân phối.
Để đảm bảo độ tin cậy và cô lập lỗi, chúng ta sẽ tổ chức các node và các cluster vào các vùng (region) các zone(khu)

cluster --> Region --> zone --> node

<a name="21"></a>
###2.1 Regions:
Swift cho phép chia cluster thành các vùng vật lý khác nhau gọi là region. Region thường được xác định bởi ranh giới địa lý. Mỗi cluster có tối thiểu 1 region nghĩa là các node đều thuộc 1 region gọi là single-region, còn khi cluster có 2 hay nhiều region thì đc gọi là multi-region (MRC)
Multi-region khi nhận được yêu cầu đọc hoặc ghi dữ liệu, đặc biệt là đọc, Swift sẽ dựa vào bản copy dữ liệu gần hơn (đo bằng độ trễ). Lựa chọn này đc gọi là lựa chọn quen thuộc (affinity)
Swift cũng có thể ghi trên affinity(những lựa chon quen thuộc) nhưng không đc mặc định Mặc định là Swift cố gắng đồng thời ghi dữ liệu tới nhiều vị trí, bất kể các region. Điều này thích hợp cho các kết nối độ trễ thấp, nơi các yêu cầu ghi, dữ liệu truyền có thể đáp ứng nhanh chóng.
Trong trường hợp kết nối có độ trễ cao, nó có thể được hỗ trợ để bật chế độ ghi vào những lựa chọn quen thuộc. Với mỗi lần ghi vào affinity, mỗi yêu cầu ghi tạo ra số lượng cần thiết của các bản sao dữ liệu địa phương, sau đó chuyển các bản sao không đc đồng bộ sang các khu vực khác. Điều này cho phép hoạt động ghi nhanh hơn, chi phí tăng khong thông thống nhất giữa các vùng cho đến khi chuyển giao không đồng hộ hoàn chỉnh.

<a name="22"></a>
###2.2 Zone:
Trong region, Swift cho phép cấu hình zone sẵn có để cô lập các lỗi. Zone sẵn có nên được định nghĩa trên các phần cứng vật lý riêng biệt để có thể cô lập lỗi từ các vùng khác. Khi đc triển khai lớn, các zone có thể đc định nhgiax là các cơ sử riêng biệt của 1 trung tâm dữ liệu, ngăn các nhau bởi firewall và cung cấp bởi các nhà cung cấp khác nhau. Khi triển khai 1 trung tâm dữ liệu duy nhất, các khu có thể là các rack khác nhau. Cần có ít nhất 1 zone cho mỗi cluster.

<a name="23"></a>
###2.3 Node:
Node là các server vật lý chạy 1 hoặc nhiều máy chủ xử lý Swift. Phần chính máy chủ Swift là proxy, account, container, object. 1 node chạy 1 account hoặc container  sẽ lưu trữ account hoặc container và metadata, 1 node sẽ chạy quá trình lưu trữ và lưu trữ dữ liệu cùng với metadata của đối tượng.
1 tập các node chạy tất cả các tiến trình của Swift lưu trữ trên hệ thống phân tán đc coi như 1 cluster, có nhiều các lưu trữ thành nhóm các node trong 1 cluster chủ yếu dựa trên các tiêu chí về vật lý. Cách để bổ sung nhóm node là dựa vào chính sách lưu trữ.

<a name="24"></a>
###2.4 Chính sách lưu trữ:
Chính sách lưu trữ là cách để các nhà khai thác Swift định nghĩa là 1 không gian bên trong các cluster có thể tùy chỉnh theo nhiều cách bao gồm về địa điểm, nhân rộng, phần cứng, partion,để đáp ứng các nhu cầu cụ thể. Như 1 công cụ mạnh mẽ và linh hoạt, chính sách lưu trữ được thiết lập ở cấp đối tượng và thực hiện ở cấp container. Điều này tăng tính linh hoạt của 1  việc lưu trữ đối tượng. Ví dụ 1 chính sách lưu trữ có thể phân phối dữ liệu trên nhiều region, có thể triển khai 1 bản sao trên 1 ổ duy nhất và bản sao khác trên ổ SSD

<a name="3"></a>
##3. Tiến trình xử lý của server
Swift chạy nhiều process để quản lý dữ liệu của nó, và mỗi node có thể chạy 1 hoặc tất cả procees đó. Để dự phòng và tăng độ bền, thì nên là nhiều node. Máy chủ xử lý chính Swift là proxy, account, container và object. Khi tiến trình proxyserver chạy trên 1 node thì node đó đc gọi là node proxy, Khi các node chạy account, container, object nó sẽ lưu trữ các loại liên quan đến dữ liệu nên được gọi là node lưu trữ, sẽ có 1 số dịch vụ khác chạy trên node lưu trữ để đảm bảo tính nhất quán dữ liệu.
Khi đề cập đến tập hợp các tiến trình chạy trên 1 node trong cluster, chúng ta nhắc đến lớp tiến trình, hay còn gọi là lớp proxy, lớp account, lớp container và lớp object

Cụ thể về tiến trình xử lý trong server
<ul>
<li><b>Proxy layer</b>: 
Lớp tiến trinh proxy là phần duy nhất của cụm Swift và dùng để giao tiếp với khách hàng bên ngoài. Nó đọc và ghi lại tọa độ yêu cầu của khách hàng và đảm bảo thực hiện đọc và ghi của hệ thống. 
Ví dụ: khi 1 khách hàng gửi yêu cầu ghi cho 1 đối tượng tới Swift, tiến trình proxy sẽ xác định các node lưu trữ dữ liệu và gửi dữ liệu đến tiến trình object trên các node kiêm nhiệm. Nếu 1 trong số các node chính đó không có sẵn tiến trình proxy server sẽ chọn 1 node thích hợp để ghi dữ liệu. Khi phần lớn tiến trình xử lý  hoàn thành, thì tiến trình proxy sẽ trả về mã success tới cho khách hàng.
Tất cả cácc tin nhắn đến và đi từ proxy đều sử dụng chuẩn HTTP như GET, PUT, POST, DELETE và các mã trả về. Tiến trình proxyserver cũng xử lý các lỗi và điều phối về mặt thời gian. Nó sử dụng kiến trúc share-nothing nên có thể thu nhỏ lại tùy thuộc vào khối lượng công việc.
</li>

<li><b>Account layer</b>: 
Các tiến trình account cung cấp metadata cho account cá nhân và 1 danh sách container có trong account đó. Nó lưu trữ thông tin trong cơ sở dữ liệu SQLite trên các ổ vật lý. Giống như dữ liệu của Swift, bản sao dữ liệu account được thực hiện và phân phối trên các cluster
</li>
<li><b>Container layer</b>: 
Các tiến trình container quản lý các metadata container và danh sách các dữ liệu object trong mỗi container đó. Các server không biết nơi lưu các đối tượng, nó chỉ biết các container lưu trữ nó. Các danh sách đối tượng được lưu trữ trong cơ sở dữ liệu SQLite. Như với tất cả các dữ liệu của Swift, bản sao của dữ liệu được lưu trữ trên các cluster khác nhau. Các tiến trình container cũng theo dõi số liệu thống kê như tổng số đối tượng và tổng dung lượng lưu trữ của container.
</li>

<li><b>Object layer</b>: 
Các tiến trình object cung cấp các điểm lưu trữ có thể được lưu trữ, truy xuất, xóa các đối tượng trên ổ cứng của các node. Đối tượng được lưu thành các file nhị phân trên các ổ cứng và sử dụng dường dẫn chứa phân vùng của nó cũng như mốc thời gian của từng dữ liệu. Điều này sẽ cho tiến trình lưu trữ nhiều phiên bản của 1 đối tượng trong khi phục vụ các phiên bản mới nhất.
Metadata của đối tượng được lưu trữ trong phần thuộc tính mở rộng của các tập tin (xattrs), được hỗ trợ bởi hầu hết các hệ thống tập tin ngày nay. Thiết kế này cho phép dữ liệu của đối tượng và metadata được lưu trữ với nhau và đc sao chép cùng với nhau.
Mỗi đối tượng được lưu trữ như 1 file đơn trên ổ đã trừ khi kích thước của nó vượt quá kích thước tối đa được cấu hình cho Swift. Giá trị kích thước mặc định là khá nhỏ, 5gb để ngăn việc 1 đối tượng có thể làm đầy 1 ổ cứng trong khi các cluster khác thì trống. Nếu 1 đối tượng lớn được lưu trữ, nó được chia thành nhiều phần và được lưu với những dấu hiệu riêng để có thể ghép lại như cũ
</li>
</ul>

<a name="4"></a>
##4. Tiến trình thống nhất
Lưu trữ  dữ liệu trên các ổ và cung cấp API cho dữ liệu là không khó. Phần khó là xử lý lỗi. Các tiến trình thống nhất (Consistaency procees) chịu trách nhiệm tìm kiếm và sửa các lỗi gây ra bởi sự hư hỏng dữ liệu hoặc hư hỏng ở phần cứng. Nó đảm bảo đực "độ bền" của Swif.
Nhiều tiến trình thống nhất được chạy ngầm để đảm bảo tính toàn vẹn dữ liệu và tính sẵn sàng của dữ liệu. Các tiến trình này chạy ngầm trên các node mà các tiến trình container, account hay object đang chạy. Các tiến trình proxy thì không liên quan đến tiến trình này vì nó không ảnh hương tới việc lưu trữ account, container hay object. Tiến trình consistency chạy phụ thuộc vào các tiến trình: 
- tiến trình account được hỗ trợ bởi các thành phần account auditor, account replicator và account reaper của tiến trình thống nhất.
- tiến trình container được hỗ trợ bởi các thành phần container auditor, container replicator, updaters, và container sync của tiến trình thống nhất
- tiến trình object được hỗ trợ bởi các thành phần object auditor, replicator object, updaters, và object expirer có trong tiến trình thống nhất.

Như bạn có thể thấy, các auditor và replicator được sử dụng ở cả 3 quá trình của việc lưu trữ. Updater thì chỉ đc sử dụng với container và object, những phần khác chỉ được sử dụng trong 1 tiến trình duy nhất. Chúng ta sẽ cùng tìm hiểu kỹ hơn về các thành phần:
- auditor: tiến trình auditor chạy ngầm trên tất cả các node lưu trữ dữ liệu. Nếu tiến trình account, container, object chạy trên 1 node nó sẽ  có những auditor tương ứng.  Những auditor account, container, object sẽ liên tục quét trên các ỏ cứng trên các nốt lưu trữ dữ liệu để đảm bảo rằng dữ liệu không bị hệ thống làm hư hỏng. Nếu phát hiện ra dữ liệu bị hư hỏng, các auditor sẽ chuyển dữ liệu đó đến khu vực kiểm tra riêng.
- replicator: Tiến trình này sẽ đảm bảo đủ các bản copy của dữ liệu mới nhất được lưu trữ trên các cluster. Điều này cho phép các cluster ở trong 1 trạng thái nhất quán khi đối mặt với các lỗi tạm thời như mất mạng hoặc hỏng ổ đĩa. Các replicator account, container, object chạy ngầm với các tiến trình tương ứng. Các tiến trình replicator sẽ liên tục kiểm tra dữ liệu đối với node local của mình với 1 node điều khiển chính trong cluster. Nếu phát hiện thấy node chính gặp vấn đề về dữ liệu như mất hoặc bị cũ, thì các replicator sẽ đẩy dữ liệu của mình ra để cập nhật hoặc thay thể dữ liệu cũ. Điều này được gọi là bản cập nhật nhân rộng.  Cần lưu ý rằng các replicator chỉ đẩy dữ liệu từ local lên remote chứ không lấy các bản dữ liệu cpoy từ các node remote về khi dữ liệu local bị hết hạn hoặc bị mất.
Replicator cũng xử lý các object hoặc các container bị xóa bỏ. Object bị xóa bằng các tạo ra 1 tập tin đánh dấu đã xóa(có tên kết thức bằng .ts) là phiên bản mới nhất của object đó. Repilcator đẩy các tập tin bia mộ này tới các bản khác như 1 bản cập nhật mới nhất và các đối tượng sẽ được gỡ bỏ trên toàn bộ hệ thống.

Việc xóa container đòi hỏi phải không có object trong đó. Dữ liệu container được đánh dấu là bị xóa và replicator đẩy các phiên bản này ra và sẽ gỡ bỏ các container trên các cluster.
- account reaper: 1 yêu cầu xóa account sẽ thiết lập cho account trạng thái đã xóa nếu nó không còn được sử dụng. Điều này cũng đánh dấu cho reaper account. Khi reaper account thấy 1 account bị đánh dấu là xóa, nó sẽ bắt đầu gỡ bỏ hết các liên kết giữa các container và object liên quan đến account đó và sau cùng là lại bỏ bản ghi account đó. Để tránh lỗi, các reaper account sẽ được cấu hình với 1 độ trễ nhất định trước khi nó bắt đầu xóa dữ liệu. 
- container, object updater: chịu trách nhiệm giữ các danh sách container tới 1 kỳ hạn. Nó sẽ cập nhật số lượng các object, số lượng container và số byte được sử dụng trong metadata account
Các updater của object cũng có trách nhiệm tương tự, đảm bảo danh sách của các container là chính xác. Tuy nhiên, các object updater được chạy như 1 sự dư thừa. Các object server chủ yếu là chịu trách nhiệm cập nhật danh sách object cho container. Nhưng các updater sẽ làm việc đó đảm bảo danh sách cho container cũng như object là số byte được sử dụng trong metadata container.
-  object expirer: chỉ định đối tượng tự động xóa sau 1 khoảng thời gian nhằm thanh lọc dữ liệu được chỉ định.

<a name="5"></a>
##5. Định vị dữ liệu.
Với các dữ liệu của account, container, object được phân phối trên nhiều node trong cluster, cta tự hỏi làm thế nào để các thành phần cảu hệ thống và dịch vụ tìm thấy 1 đoạn bất kỳ của dữ liệu trong cluster?
Câu trả lời ngắn gọn cho điều này là khi 1 tiến tình chạy trên các node cần tìm 1 phần của account, container, object nó sẽ đi tới bản copy của riêng và tìm tất cả các vị trí của account ring, container rings,hoặc 1 trong các object rings, Swift lưu trữ nhiều bản sao của từng đoạn dữ liệu, do đó khi nhìn vào nơi dữ liệu được lưu trữ sẽ trả về nhiều vị tí khác nhau. Sau đó tiến trình có thể tiến hành truy vấn vị trí của dữ liệu.
Ring ở đây là tập hợp các bảng tra cứu phân phối tất cả các node trong cluster. Swift sử dụng 1 phiên bản băm thích hợp cho rings. Để hiểu được cách rings được tạo ra và cách chúng được sử dụng cho các vị trí lưu trữ trữ dữ liệu, đầu tiên ra cần xem xét về các hàm băm và các hàm băm rings phù hợp. sau đó chúng ta sẽ xem xét sửa đổi hàm băm rings phù hợp và cách rings được tạo ra.

<a name="51"></a>
###5.1 Cơ bản Ring: Hàm băm
Trước khi nhìn cụ thể về hàm băm phù hợp với Swift chúng ta sẽ nhìn hàm băm cơ bản 1 cách tổng quan vầ làm thế nào chúng ta có thể sử dụng hàm băm để xác định việc lưu trữ trên ổ đĩa. Hàm băm có thể hiều là 1 phương pháp mà lấy chỗi dài của dữ liệu vào và tạo ra 1 chiếu tham chiếu ngắn hơn cho nó. 1 điểm quan trọng là nó luôn luôn trả về kết quả giống nhau.
Một phương pháp tương đối đơn giản của việc xác định nơi để lưu trữ đối tượng là sử dụng các thuật toán MD5 để băm vị trí các đối tượng lưu trữ và sau đó lấy giá trị băm được chia cho số lượng ổ đĩa sẽ ra được ổ đĩa lưu trữ đối tượng.
Ví dụ
md5 ( /account/container/object) =  f9db0f833f1545be2e40f387d6c271de
đổi ra hệ 10 sẽ là 332115198597019796159838990710599741918
nếu dùng 4 ổ đĩa 0,1,2,3 lấy giá trị trên chia cho 4 dư là 2 vậy object sẽ được lưu vào ổ 2 (stt=3)
Hạn chế của phương pháp này sẽ là nếu thêm hay bớt 1 ổ thì vị trí lưu sau khi kiểm tra sẽ không còn chính xác. Sẽ phải tính toán lại toàn bộ việc lưu trữ dữ liệu trên clusster khiến cho tăng trafice mạng và khiến dữ liệu không còn khả dụng

<a name="52"></a>
###5.2 Cơ bản Rings: Hàm băm phù hợp.
<img src="http://i.imgur.com/R4RRA84.png">
Hàm băm phù hợp sẽ giảm thiểu số lượng đối tượng di chuyển khi ta thêm hoặc bớt ổ cứng trong 1 cluster. Thay vì gán vào 1 giá trị trực tiếp của 1 ổ đĩa (ID ổ)  thì 1 loạt các giá trị sẽ được kết nối vs 1 ổ đĩa. Điều này thực hiện bằng cách lập bản đồ tất cả các giá trị hash có thể thành 1 vòng tròn, sau đó mỗi ổ đĩa sẽ được gán cho 1 điểm trên vòng tròn đó dựa trên 1 giá trị băm của ổ đĩa. (băm ip ô, băm tên ổ...) kết quả là các ổ đĩa sẽ được đặt trong 1 trật tự khá ngẫu nhiên xung quanh 1 vòng tròn. Khi đối tượng cần được lưu trữ, giá trị băm của đối tượng được xác định và nằm trên đường tròn. Sau đó hệ thống sẽ quay theo chiều kim đồng hồ tìm kiếm xung quanh vòng tròn để xác định vị trí các điểm đánh dấu ổ đĩa gần nhất, đây sẽ là ổ đĩa mà đối tượng được đặt. Khi ta thêm 1 ổ đĩa mới, hoặc loại bỏ 1 ổ đĩa thì chỉ 1 vài đối tượng bị ảnh hưởng. Khi 1 ổ đĩa được thêm vào vòng tròn, ổ đĩa tiếp theo theo chiều kim đồng hồ sẽ bị mất đi đối tượng bất kỳ thuộc ổ cũ, những đối tượng này sẽ được sao chép sang ổ mới, phần còn lại ở ổ cũ sẽ vẫn được giữ nguyên, ĐIều này hiệu quản hơn nhiều so với cách cũ.
Đây là cách đơn giản với mỗi ổ đĩa có 1 điểm đánh dấu. Trong thực tế thì mỗi ổ đĩa có r nhiều dấu trên 1 vòng tròn chứ không chỉ 1 dấu. Hầu hết các hàm băm rings có thể đánh dấu r nhiều thậm chí lên tới hàng trăm đấu cho 1 ổ trên 1 vòng tròn, nó sẽ giúp lưu trữ được nhiều đối tượng hơn vào 1 ổ đĩa, và đối tượng cũng sẽ được hạn chế dịch chuyển hơn.

<a name></a>
###5.3 Rings: Sửa đổi băm phù hợp:
<img src="http://i.imgur.com/Rx2XUdd.png">
Swift sử dụng Hàm băm phù hợp đã sửa đổi. Những thay đổi này tạo ra 1 rings băm phù hợp với phân vùng sử dụng số bản sao, khóa bản sao, và cơ chế phân phối dữ liệu(như trọng lượng ổ, và vị trí duy nhất có thể) để xây dựng vị trí lưu trữ cá nhân. 1 account ring sẽ được tạo ra cho 1 cluster và được sử dụng để xác định vị trí dữ liệu account. 1 container ring được tạo ra cho 1 cluster và được sử dụng để xác định vị trí container. 1 rings chính sách lưu trữ đối tượng sẽ được tạo cho mỗi chính sách lưu trữ và sự dụng để xác định vị trí dữ liệu.

####Partion:
Với hàm băm rings chưa sửa đổi khi thêm bớt ổ, nhiều dãy băm sẽ trở nên to hơn hoặc nhỏ đi so với khi ổ đĩa được gắn thêm hoặc bớt đi, những khoảng này có thể dẫn đến việc đối tượng không sẵn sàng khi nó đang di chuyển trong việc thây đổi vị trí ổ. Để ngăn điều này, Swift tiếp cận vòng băm khác nhau, mặc dù ringss vẫn chia thành nhiều dãy băm nhỏ nhưng các dải này sẽ có chung kích thước và sẽ không thay đổi về số lượng. Những vùng này sẽ được fix cố định sau đó được gán cho ổ đĩa bằng thuật toán sắp xếp. Đi vào chi tiết các phần được tính toán như nào và cách các ổ đĩa đc gắn cho chúng.
- partion power: 
 ```sh
 tổng số phân vùng (partion) trong cluster = 2^(partion power)
```
số lượng driver có thể thay đổi nhưng số partition thì không

- <b>Replica count</b>: là số bản sao của từng partion được đặt trên cluster.Ví dụ: nếu bạn có Replica count=3 nghĩa là mỗi partion sẽ được nhân lên 3 lần. Mỗi bản này sẽ được lưu trữ trên các thiết bị khác nhau. Khi object được đưa vào cluster, giá trị băm sẽ phù hợp với 1 partion và được sao chép vào cả 3 vị trí của partion đó (Ngĩa là khi chia thành các partion thì partion cũng đc sao lưu thành 3 lần, dữ liệu vào partion cũng được sao lưu theo partion đó???)
càng nhiều bản sao càng bảo vệ bạn khỏi việc mất dữ liệu khi xảu ra lỗi(đặc biệt các bản sao ở các vị trí địa lý hoặc trung tâm dữ liệu khác nhau). Replica count là số thực thường mặc định là 3.0. trong 1 vài trường hợp cá biệt, ví dụ 3,15 thì nghĩa là 15% số ổ sẽ có thêm 1 bản sao, tổng cộng là 4. Giúp cho việc thay đổi dần dần từ số nguyên này sang số nguyên khác mà không gây bão hòa mạng.
Partion replication còn bao gồm ổ đĩa mà nó bàn giao. Nó nghĩa là khi 1 ổ đĩa bị hỏng, các quá trình (replicationg và auditing) sẽ thông báo và đẩy dữ liệu ra khỏi vị trí này,. Điều này làm giảm đáng kể thời gian sửa chữa so với RAID hoặc three-way mirroring. Xác suất mà tất cả các phân vùng nhân rộng trong hệ thống đều hư hỏng trước khi cluster thông báo và kéo dữ liệu ra khỏi đó là rất nhỏ, đó là lý do đảm bảo độ bền của Swift, Tùy thuộc vào việc đảm bảo độ bền mà bạn yêu cầu và việc thất bạn phân tích của thiết bị lưu trữ của bạn, bạn có thể thiết lạp số bản sao hợp lý với nhu cầu mà bạn cần sử dụng

- <b>Replica locks</b>: Trong khi 1 partion được di chuyển, thay đổi, Swift sẽ khóa bản sao của phân vùng đó để nó không đủ điều kiện để được di chuyển đảm bảo dữ liệu sẵn có. Khóa được sử dụng cả khi rings được cập nhật cũng như hoạt động khi dữ liệu được thay đổi. Độ dài chính xác của thời gian để khóa các partion được thiết lập bởi trường "min_part_hours"

<a name="6"></a>
##6. Phân phối dữ liệu:
Ngoài việc sử dụng cấu trúc rings đã sửa đổi. Swift còn có 2 cơ chế khác để phân phối dữ liệu 1 cách đồng đều.
- Trọng số: Swift sử dụng giá trị gọi là trọng số cho mỗi thiết bị trong cluster. Giá trị do người dụng định nghĩa ra được thiết lập khi ổ được add thêm vào và được so với các trọn số của các ổ có sẵn trong rings. Trọng số này giúp cluster tính toán có bao nhiêu partion nên được gán cho ổ đĩa. Trọng số cao hơn thì partion nhiều hơn
- unique as possible: Để đảm bảo các clusster lưu trữ dữ liệu là đồng đều trên các không gian đã được định nghĩa (region, zone, node và disk). Partion Swift sử dụng thuật toán gọi là unique as possible. Thuật toán này sẽ xác định dựa trên tiêu chí ít được sử dụng nhất region, zone, máy chủ(ip:port), và sau đó nếu cần thiết, ổ đĩa ít sử dụng nhất và đó là vị trí partion được đặt. Công thức ít sử dụng nhất cũng cố gắng đặt các bản sao partion càng xa nhau càng tốt. Vị trí tính toán này cho phép sử dụng deployers 1 cách linh hoạt trong việc tổ chức cơ sở hạ tầng của họ khi sử dụng cấu hình Swift để tận dụng lợi thế của những j đã được triển khau thay vi bắt các deloyers thay đổi phần cứng đẻ phù hợp với ứng dụng.

<a name="7"></a>
##7. Tạo và cập nhật Rings
Chúng ta sẽ nhìn cụ thể vào cách tạo rings account, rings conatiner, rings object và tìm hiểu về cấu trúc nội bộ cunar các rings.
Rings được tạo ra và cấp nhật bởi ring-builder. Có 2 bược trong việc tạo ra rings: cập nhật các file builder và cân bầng các rings.
- Tạo và cập nhật các builder file: trong quá trình tạo cluster, Swift sử dụng các tiện ích ring-builder để tạo các file biulder, đó là bản thiết kê ngắn cho việc tạo ra rings. Các file builder riêng biệt được tạo cho các account, container và tứng chính sách lưu trữ object, nó chứa thông tin như partion power, repica count, replica lock time và vị trí của ổ trong cluster. Swift sử dụng ring-builder để cập nhật các file builder với các thông tin cập nhật  trong ổ đĩa.
Lưu ý quan trọng là thông tin trong mỗi file builder này là tách biệt với các file khác. Ví dụ như có thể thêm 1 số ổ đĩa cho account và conatainer nhưng không được cho object
- Cân bằng rings: Khi các builder file được tạo hoặc cập nhật với tất cả thông tin cần thiết, rings sẽ được tạo. Tiện ích rings-builder chạy sử dụng các lệnh rebalance với builder file là tham số đầu vào. Điều này được thực hiện cho mỗi rings: accouunt, container, object. Mỗi khi rings được tạo , nó phải sao chép tới tất cả các node, thay thế cho những  rings cũ.


Bên trong Rings
Mỗi lần Swift xây dựng rings, 2 cấu trúc dữ liệu nội bộ quan trọng được tạo ra và truyền vào trong file builder:
- Devices list: Ring-builder chứa danh sách tất cả thiết bị mà chúng ta muốn thêm và file ring-builder. Mỗi ổ bao gồm các thông số ID, zone, trọng số, địa chỉ IP, port và tên thiết bị.
- Bảng tra cứu thiết bị: Bảng này chứa 1 hàng cho mỗi bản sao và cột là cho mỗi partion trong cluster, thường là có 3 hàng và hàng ngàn cột. Ring-bui;dẻ tính toán các ổ đĩa tối ưu để đặt mỗi bản sao phân vùng trên ổ đĩa bằng cách sử dụng các trngj số và thuật toán sắp xế unique-as-posible. Sau đó ghi lại các ổ vào trong bảng

Những cấu trúc bên trong là những ghì được gọi bở bất kỳ tiến trình Swift nào, nơi mà dữ liệu được lưu trữ. Các tiến trình Swift sẽ tính toán hàm bằm của dữ liệu mà nó đang tìm kiếm, Giá trị băm này sau đó sẽ được ánh xạ tơi 1 partion, mà có thể được nhìn thấy trong các cột ở bảng tra cứu thiết bị. Tiến trình lời gọi sẽ kiểm tra mỗi hàng replica tại cột cụ thể để xem những j lưu trữ trên ổ đĩa, sau đó nó lần thứ 2 tìm kiếm về thông tin thiết bị mà nó cần và gọi tất cả dữ liệu
