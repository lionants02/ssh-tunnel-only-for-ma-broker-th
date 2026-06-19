- [ตัวอย่าง SSH Tunnel ไปยัง MongoDB](#ตัวอย่าง-ssh-tunnel-ไปยัง-mongodb)
  - [สร้าง tunnel](#สร้าง-tunnel)
  - [เชื่อมต่อ MongoDB ผ่าน tunnel](#เชื่อมต่อ-mongodb-ผ่าน-tunnel)
  - [สร้าง user แบบอ่านอย่างเดียวใน database ที่ต้องการ](#สร้าง-user-แบบอ่านอย่างเดียวใน-database-ที่ต้องการ)
  - [ใช้ local port อื่นเมื่อเครื่อง local มี MongoDB อยู่แล้ว](#ใช้-local-port-อื่นเมื่อเครื่อง-local-มี-mongodb-อยู่แล้ว)
  - [รัน tunnel แบบ background](#รัน-tunnel-แบบ-background)
  - [ปัญหาที่พบบ่อย](#ปัญหาที่พบบ่อย)

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

## สร้าง user แบบอ่านอย่างเดียวใน database ที่ต้องการ

ให้เชื่อมต่อด้วย user ที่มีสิทธิ์สร้าง user ก่อน เช่น `root` หรือ user ที่มี role `userAdmin` บน database นั้น

```bash
mongosh "mongodb://admin_user@127.0.0.1:27017/admin" --password
```

จากนั้นสร้าง read-only user ใน database ที่ต้องการ เช่น `app_db`

```javascript
use app_db

db.createUser({
  user: "app_readonly",
  pwd: passwordPrompt(),
  roles: [
    { role: "read", db: "app_db" }
  ]
})
```

ทดสอบเชื่อมต่อด้วย user ที่สร้างใหม่

```bash
mongosh "mongodb://app_readonly@127.0.0.1:27017/app_db?authSource=app_db" --password
```

หมายเหตุ:

- `role: "read"` ทำให้ user อ่านข้อมูลใน database นั้นได้ แต่ไม่สามารถแก้ไขข้อมูลได้
- ถ้าสร้าง user ไว้ใน database อื่น ให้เปลี่ยน `authSource` ให้ตรงกับ database ที่ใช้สร้าง user
- ถ้าใช้ local port อื่น เช่น `127017` ให้เปลี่ยน `127.0.0.1:27017` เป็น `127.0.0.1:127017`

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
