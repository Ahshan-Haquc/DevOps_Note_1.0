# 💾 Docker Volumes for MERN Stack Developers

## Docker Volume কী?

Docker Volume হলো Container-এর Data Permanently সংরক্ষণ করার একটি পদ্ধতি।

সহজ ভাষায়,

> Container Delete হলেও যাতে Data Delete না হয়, সেই ব্যবস্থাই হলো Docker Volume।

---

# 🤔 Volume কেন দরকার?

ধরো তুমি MongoDB Container চালালে:

```bash
docker run -d --name mongodb mongo
```

এখন Database-এ কিছু Data Save করলে:

```text
Users
Products
Orders
```

সব Data Container-এর ভিতরে থাকবে।

কিন্তু যদি Container Delete করো:

```bash
docker rm mongodb
```

তাহলে Database-এর সব Data হারিয়ে যাবে। ❌

---

# Volume ব্যবহার করলে কী হয়?

```text
MongoDB Container
        │
        ▼
Docker Volume
        │
        ▼
Persistent Data
```

এখন Container Delete হলেও Data থাকবে। ✅

---

# Volume Create

```bash
docker volume create mongo-data
```

Volume List দেখতে:

```bash
docker volume ls
```

Output:

```text
DRIVER    VOLUME NAME

local     mongo-data
```

---

# Volume ব্যবহার করে MongoDB Run

```bash
docker run -d \
--name mongodb \
-v mongo-data:/data/db \
mongo
```

এখানে,

```text
mongo-data
```

হলো Docker Volume।

```text
/data/db
```

হলো MongoDB Container-এর Data Folder।

---

# Real MERN Example

```text
Backend Container
        │
        ▼
MongoDB Container
        │
        ▼
Mongo Volume
```

এখন:

```bash
docker rm mongodb
```

করলেও Database Data হারাবে না।

নতুন Container Run করলে আগের Data আবার পাবে।

---

# Host Folder Mount (Development)

কখনো কখনো Host Machine-এর Folder Container-এর সাথে Connect করতে হয়।

উদাহরণ:

```bash
docker run -v ./uploads:/app/uploads my-app
```

এখানে,

```text
./uploads
```

Host Machine-এর Folder।

```text
/app/uploads
```

Container-এর Folder।

---

# MERN Developer হিসেবে সবচেয়ে বেশি কোথায় Volume ব্যবহার করবে?

### MongoDB Data

```bash
-v mongo-data:/data/db
```

---

### Uploaded Images

```bash
-v uploads:/app/uploads
```

---

### Log Files

```bash
-v logs:/app/logs
```

---

# Volume Commands

Volume Create:

```bash
docker volume create my-volume
```

Volume List:

```bash
docker volume ls
```

Volume Details:

```bash
docker volume inspect my-volume
```

Volume Delete:

```bash
docker volume rm my-volume
```

---

# Volume না ব্যবহার করলে

```text
MongoDB Container

↓

Data Stored

↓

Container Deleted

↓

Data Lost ❌
```

---

# Volume ব্যবহার করলে

```text
MongoDB Container

↓

Docker Volume

↓

Data Stored

↓

Container Deleted

↓

Data Still Exists ✅
```

---

# Final Summary

```text
Container = Temporary

Volume = Permanent Storage
```

একজন MERN Developer হিসেবে Volume-এর সবচেয়ে গুরুত্বপূর্ণ ব্যবহার হলো MongoDB Data Persist করা। Production Environment-এ Database Container Volume ছাড়া Run করা উচিত নয়।
