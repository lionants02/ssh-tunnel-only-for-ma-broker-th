# ตัวอย่าง SSH Tunnel ไปยัง PostgreSQL

ตัวอย่างนี้ใช้ SSH broker เป็นทางผ่านไปยัง PostgreSQL ที่อยู่บนเครื่อง `10.10.20.16` port `5432`

## สร้าง tunnel

```bash
ssh -i ~/.ssh/ma_tunnel_key -N \
  -L 127.0.0.1:15432:10.10.20.16:5432 \
  ma-tunnel@broker.example.com
```

ความหมายของ option ที่สำคัญ:

- `-i ~/.ssh/ma_tunnel_key` ใช้ private key สำหรับ tunnel user
- `-N` ไม่เปิด remote command หรือ shell
- `-L 127.0.0.1:15432:10.10.20.16:5432` เปิด local port `15432` แล้วส่งต่อไป PostgreSQL ปลายทางผ่าน SSH broker

## เชื่อมต่อ PostgreSQL ผ่าน tunnel

```bash
psql -h 127.0.0.1 -p 15432 -U app_user -d app_db
```

หรือใช้ connection string

```bash
psql "postgresql://app_user@127.0.0.1:15432/app_db"
```

ถ้า application ต้องใช้ connection string ให้ชี้ host เป็น `127.0.0.1` และ port เป็น local port ที่เปิดไว้

```text
postgresql://app_user:CHANGE_ME@127.0.0.1:15432/app_db
```

## รัน tunnel แบบ background

```bash
ssh -i ~/.ssh/ma_tunnel_key -fN \
  -L 127.0.0.1:15432:10.10.20.16:5432 \
  ma-tunnel@broker.example.com
```

เมื่อต้องการปิด tunnel ให้หา process และหยุดเอง

```bash
ps aux | grep "127.0.0.1:15432:10.10.20.16:5432"
```

## เปิด MongoDB และ PostgreSQL ใน SSH connection เดียว

```bash
ssh -i ~/.ssh/ma_tunnel_key -N \
  -L 127.0.0.1:27017:10.10.20.15:27017 \
  -L 127.0.0.1:15432:10.10.20.16:5432 \
  ma-tunnel@broker.example.com
```

## ปัญหาที่พบบ่อย

- ถ้าเจอ `channel open failed: administratively prohibited` ให้ตรวจสอบ `permitopen` ใน authorized key และ `PermitOpen` ใน `sshd_config`
- ถ้า connect ได้แต่ PostgreSQL login ไม่ผ่าน ให้ตรวจสอบ username, database name, password และ policy ใน `pg_hba.conf`
- ถ้า local port `15432` ถูกใช้แล้ว ให้เปลี่ยน port ฝั่งซ้ายของ `-L`
