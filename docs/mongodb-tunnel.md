# ตัวอย่าง SSH Tunnel ไปยัง MongoDB

ตัวอย่างนี้ใช้ SSH broker เป็นทางผ่านไปยัง MongoDB ที่อยู่บนเครื่อง `10.10.20.15` port `27017`

## สร้าง tunnel

```bash
ssh -i ~/.ssh/ma_tunnel_key -N \
  -L 127.0.0.1:27017:10.10.20.15:27017 \
  ma-tunnel@broker.example.com
```

ความหมายของ option ที่สำคัญ:

- `-i ~/.ssh/ma_tunnel_key` ใช้ private key สำหรับ tunnel user
- `-N` ไม่เปิด remote command หรือ shell
- `-L 127.0.0.1:27017:10.10.20.15:27017` เปิด local port `27017` แล้วส่งต่อไป MongoDB ปลายทางผ่าน SSH broker

## เชื่อมต่อ MongoDB ผ่าน tunnel

```bash
mongosh "mongodb://127.0.0.1:27017"
```

ถ้า MongoDB ต้องใช้ username/password

```bash
mongosh "mongodb://db_user@127.0.0.1:27017/app_db?authSource=admin"
```

หรือถ้าต้องการระบุ password ผ่าน prompt

```bash
mongosh "mongodb://db_user@127.0.0.1:27017/app_db?authSource=admin" --password
```

## ใช้ local port อื่นเมื่อเครื่อง local มี MongoDB อยู่แล้ว

ถ้า port `27017` บนเครื่อง local ถูกใช้งานอยู่ ให้เปลี่ยน local port เป็น `127017`

```bash
ssh -i ~/.ssh/ma_tunnel_key -N \
  -L 127.0.0.1:127017:10.10.20.15:27017 \
  ma-tunnel@broker.example.com
```

แล้วเชื่อมต่อด้วย

```bash
mongosh "mongodb://127.0.0.1:127017"
```

## รัน tunnel แบบ background

```bash
ssh -i ~/.ssh/ma_tunnel_key -fN \
  -L 127.0.0.1:27017:10.10.20.15:27017 \
  ma-tunnel@broker.example.com
```

เมื่อต้องการปิด tunnel ให้หา process และหยุดเอง

```bash
ps aux | grep "127.0.0.1:27017:10.10.20.15:27017"
```

## ปัญหาที่พบบ่อย

- ถ้าเจอ `channel open failed: administratively prohibited` ให้ตรวจสอบ `permitopen` ใน authorized key และ `PermitOpen` ใน `sshd_config`
- ถ้า connect ได้แต่ MongoDB login ไม่ผ่าน ให้ตรวจสอบ credential และ `authSource`
- ถ้า local port ถูกใช้แล้ว ให้เปลี่ยน port ฝั่งซ้ายของ `-L`
