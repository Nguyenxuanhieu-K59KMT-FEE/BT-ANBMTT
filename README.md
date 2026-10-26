## Cách chạy

Yêu cầu: Windows, .NET SDK (phiên bản khớp với TargetFramework trong file .csproj).

dotnet run --project ATBM_App -c Release

Nên chạy bản Release để số liệu đo tốc độ chính xác.

## Bài 1: Mã hoá AES (đối xứng)

1. Sinh khoá, mã hoá và giải mã văn bản

<img width="1920" height="1080" alt="1793002878317_118498820250161574_4718576041742126713_fd7f473ea784c1f4e131cd53b2b38488" src="https://github.com/user-attachments/assets/7c4e1e84-8d37-4c73-b4f3-a06970001e41" />



AES văn bản

Sinh khoá AES-256 (hiển thị dạng base64), mã hoá một đoạn văn bản ra chuỗi base64 (IV + ciphertext) rồi giải mã lại ra đúng bản rõ. Ảnh cũng cho thấy chương trình báo lỗi "Chưa có khoá AES" khi bấm mã hoá mà chưa sinh hoặc import khoá.


2. Lưu khoá ra file và import khoá từ file

<img width="1920" height="1080" alt="1793002878318_118498820250161574_4718576041742126713_d421196b73aa083f017db57fc55e5197" src="https://github.com/user-attachments/assets/0a2a1ed4-f7cb-4603-ad45-e71a7229ee78" />

AES lưu và import khoá

Khoá được lưu ra file aes.key (một dòng base64), sau đó import lại từ file (báo "Đã import khoá 256 bit"). Dùng khoá vừa import để mã hoá và giải mã văn bản vẫn cho kết quả đúng, chứng tỏ việc lưu và import khoá hoạt động chính xác.

3. Thử nghiệm file 1 GB: thời gian mã hoá, giải mã và so sánh byte
<img width="680" height="205" alt="1793002878319_118498820250161574_4718576041742126713_ebdf5aaf5570bde2873be6d275789519" src="https://github.com/user-attachments/assets/e3b09b31-2b2c-4b5b-b845-76f46fcd538a" />



AES benchmark 1 GB

Tạo file test 1024 MB (1.073.741.824 byte), sau đó chạy benchmark trọn gói gồm mã hoá, giải mã và so sánh byte giữa file nguồn với file sau giải mã.

| Chỉ số	Kết quả |
| --- |
| Mã hoá	1,127 s (908,7 MB/s) |
| Giải mã	0,687 s (1489,8 MB/s) |
| So sánh byte	0,655 s, giống nhau 100% (cả hai file đều 1.073.741.824 byte) |

Nhận xét: thuật toán đúng (file giải mã trùng khớp từng byte với file gốc). Tốc độ đạt hàng trăm đến hàng nghìn MB/s nhờ AES phần cứng, giải mã nhanh hơn mã hoá vì CBC giải mã có thể song song hoá.

## Bài 2: Mã hoá RSA (bất đối xứng)


4. Sinh cặp khoá, xuất PEM và xem cấu trúc PEM


<img width="1920" height="1080" alt="1793002878320_118498820250161574_4718576041742126713_0c16e4fb2162e31ed404cc21582c165c" src="https://github.com/user-attachments/assets/443ac401-46dd-4465-bd56-4296e1b972a4" />

 RSA xem PEM

Sinh cặp khoá RSA-2048, xuất ra public.pem và private.pem. Chức năng Xem PEM hiển thị nội dung PEM và giải thích cấu trúc byte theo ASN.1 DER (Tag-Length-Value): khoá công khai dài 294 byte gồm SEQUENCE > OID rsaEncryption, NULL, BIT STRING > SEQUENCE{ n (257 byte), e = 010001 }. Khoá riêng dài 1190 byte, bắt đầu bằng SEQUENCE với version, n, e...

5. Nhập PEM, đọc các thành phần của khoá



<img width="1920" height="1080" alt="1793002878321_118498820250161574_4718576041742126713_6206345a76374f00e9275a8a1e27bd10" src="https://github.com/user-attachments/assets/809643fd-167e-4882-a4cd-1d3c66c97e33" />


RSA nhập PEM

Phần cuối cấu trúc khoá riêng PKCS#1 (d, p, q, dP, dQ, qInv) và kết quả nhập PEM: file public cho ra n (2048 bit) và e = 65537; file private cho ra thêm d, p, q, dP, dQ, qInv.

Các khoá trong ảnh chỉ là khoá thử nghiệm, không dùng cho mục đích thật.

6. Mã hoá khoá 32 byte bằng public key, giải mã bằng private key

RSA mã hoá giải mã

<img width="1920" height="1080" alt="1793002878322_118498820250161574_4718576041742126713_9598ef53a6643a42daec1fbaf067569a" src="https://github.com/user-attachments/assets/5eff8165-2707-4dd0-b209-07807a46c149" />



Mã hoá chuỗi base64 của một khoá 32 byte bằng public.pem, lưu bản mã ra file (base64), rồi giải mã bằng private.pem. Kết quả giải mã trùng khớp với dữ liệu gốc.

7. Kiểm tra giới hạn dữ liệu 190 byte

RSA giới hạn 190 byte

<img width="1920" height="1080" alt="1793002878323_118498820250161574_4718576041742126713_30e1ef02fc985754ecece259e3b51920" src="https://github.com/user-attachments/assets/a93f2ba8-607f-44bb-8f33-d3fab2d3f7bb" />


Với dữ liệu 191 byte, chương trình báo lỗi "Dữ liệu 191 byte > tối đa 190 byte (OAEP-SHA256)". Đây là giới hạn của RSA-2048 với OAEP-SHA256: 256 - 2×32 - 2 = 190 byte.

8. Xuất cặp khoá ra thư mục riêng và mã hoá, giải mã lại

RSA khoá trong thư mục riêng

<img width="1920" height="1080" alt="1793002878324_118498820250161574_4718576041742126713_fb77d20c0b7b1be6f0fcc9416705b210" src="https://github.com/user-attachments/assets/8f860dff-1b31-4c3b-a57c-beecd13283a5" />


Cặp khoá được xuất vào thư mục riêng khóa A. Mã hoá 32 byte bằng public key trong thư mục này rồi giải mã bằng private key cùng cặp, cho kết quả trùng khớp với dữ liệu gốc.

9. Đánh giá tốc độ: public key (e nhỏ) so với private key (d lớn)

RSA tốc độ
<img width="1920" height="1080" alt="1793002878325_118498820250161574_4718576041742126713_f555310382cb77b53f85e31c228499cc" src="https://github.com/user-attachments/assets/2c7d25a2-4cd9-4ae8-b433-1f29a0aeb395" />



| Độ dài khoá	Mã hoá (public)	Giải mã (private)	Private chậm hơn |
| --- |
| 1024 bit	0,0060 ms	0,1081 ms	18,1 lần |
| 2048 bit	0,0208 ms	0,6262 ms	30,1 lần |
| 3072 bit	0,0413 ms	1,9257 ms	46,6 lần |
| 4096 bit	0,0684 ms	4,2478 ms	62,1 lần |

Nhận xét: mã hoá bằng public key (e = 65537 chỉ khoảng 17 bit) nhanh hơn giải mã bằng private key (d dài cỡ bằng n) hàng chục lần, và khoảng cách càng tăng khi khoá dài hơn. Vì vậy RSA chỉ dùng cho dữ liệu nhỏ (khoá AES, giá trị băm), còn dữ liệu lớn được mã hoá bằng AES.

SHA-256
10. Hiệu ứng thác đổ của hàm băm

SHA-256

<img width="1920" height="1080" alt="1793002878326_118498820250161574_4718576041742126713_b13767c4fd0e2637d5ee86f9a4f76c57" src="https://github.com/user-attachments/assets/6942f7af-c49b-4e9a-a465-4626d5b58862" />



Nhập hai chuỗi chỉ khác nhau rất ít (Hello world và Hello worle), chương trình tự tính SHA-256 của cả hai và đếm số bit băm khác nhau: 142/256 bit (55,5%). Lý thuyết là khoảng 50%, tức chỉ cần đổi đầu vào rất ít thì băm đổi gần như hoàn toàn (hiệu ứng thác đổ).

Chữ ký số

Chữ ký số là RSA dùng khoá bí mật trên giá trị băm của dữ liệu.

11. Cùng dữ liệu, cùng khoá nhưng chữ ký khác nhau

Chữ ký số

<img width="1920" height="1080" alt="1793002878327_118498820250161574_4718576041742126713_ea9737f2af0974489bc54f17670356a1" src="https://github.com/user-attachments/assets/edf60915-d200-4fc6-8f85-ef8b00afe661" />



Ký 3 lần cùng dữ liệu hợp đồng 001 bằng cùng một private key:

Kiểu đệm	Kết quả	Xác thực
PKCS#1 v1.5 (tất định)	3 chữ ký giống hệt nhau (1/3 chữ ký khác nhau)	True cả 3 lần
RSASSA-PSS (salt ngẫu nhiên)	3 chữ ký khác nhau (3/3)	True cả 3 lần

Phương pháp đề xuất: dùng RSASSA-PSS. Mỗi lần ký sinh một salt ngẫu nhiên đưa vào giá trị được ký, nên cùng dữ liệu, cùng người ký vẫn cho chữ ký khác nhau, nhưng khi xác thực vẫn đúng vì salt được nhúng sẵn trong chữ ký.
