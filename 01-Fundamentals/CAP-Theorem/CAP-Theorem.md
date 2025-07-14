# CAP Theorem

**Tags**: #fundamental #distributed-systems #theory
**Date**: 2024-01-01

## 📝 Overview

CAP Theorem (còn gọi là Brewer's theorem) là một khái niệm quan trọng trong thiết kế hệ thống phân tán. Định lý này phát biểu rằng một hệ thống phân tán không thể đồng thời đảm bảo cả 3 tính chất sau:

## 🎯 Ba tính chất của CAP

### **C - Consistency (Tính nhất quán)**
- Tất cả các node đều thấy cùng một dữ liệu cùng một lúc
- Mọi read operation đều nhận được write gần nhất hoặc error
- Ví dụ: Khi update balance tài khoản, tất cả users đều thấy số dư mới ngay lập tức

### **A - Availability (Tính khả dụng)**
- Hệ thống vẫn hoạt động và response requests ngay cả khi một số node bị lỗi
- Mọi request đều nhận được response (không phải error)
- Ví dụ: Website vẫn accessible ngay cả khi một số server bị down

### **P - Partition Tolerance (Khả năng chịu phân vùng)**
- Hệ thống tiếp tục hoạt động mặc dù network failures giữa các node
- System có thể survive việc mất kết nối giữa các components
- Ví dụ: Hệ thống vẫn hoạt động khi network link giữa 2 data centers bị đứt

## ⚖️ Trade-offs

Theo CAP theorem, bạn chỉ có thể chọn **2 trong 3** tính chất:

### **CP (Consistency + Partition Tolerance)**
- Ưu tiên consistency hơn availability
- Khi có network partition, system sẽ từ chối serve requests
- **Ví dụ**: Banking systems, ACID databases
- **Technologies**: HBase, MongoDB (strong consistency mode), Redis Cluster

### **AP (Availability + Partition Tolerance)**
- Ưu tiên availability hơn consistency
- System vẫn serve requests nhưng data có thể không consistent
- **Ví dụ**: Social media feeds, DNS, web caches
- **Technologies**: Cassandra, DynamoDB, CouchDB

### **CA (Consistency + Availability)**
- Chỉ hoạt động trong single node hoặc network không bao giờ bị partition
- Thực tế không khả thi cho distributed systems
- **Ví dụ**: Traditional RDBMS trong single datacenter
- **Technologies**: PostgreSQL, MySQL (single instance)

## 🎯 Thực tế trong System Design

### Network Partitions là không thể tránh khỏi
- Hardware failures
- Network congestion
- Software bugs
- Configuration errors

→ **Partition Tolerance là bắt buộc** cho distributed systems

### Lựa chọn thực tế: CP vs AP
```
Nếu chọn CP:
- Sacrifice availability during partitions
- Ensure data consistency
- Good for financial systems

Nếu chọn AP:  
- Sacrifice consistency during partitions
- Ensure system availability
- Good for content systems
```

## 🔍 Ví dụ thực tế

### **CP System - Banking**
```
Khi network partition xảy ra:
- ATM sẽ reject transactions
- Đảm bảo không có double spending
- Prefer correctness over availability
```

### **AP System - Social Media**
```
Khi network partition xảy ra:
- Users vẫn có thể post/read
- Một số posts có thể bị duplicate
- Prefer user experience over perfect consistency
```

## 💡 Beyond CAP

### **PACELC Theorem**
- **PAC**: If Partition, choose between Availability and Consistency
- **ELC**: Else (no partition), choose between Latency and Consistency

### **Eventual Consistency**
- AP systems thường implement eventual consistency
- Data sẽ consistent sau một thời gian
- Trade immediate consistency cho availability

## 🔗 Related Topics

- [[ACID Properties]]
- [[Consistency Models]]
- [[Distributed Systems]]
- [[Database Replication]]

## 📚 Further Reading

- Eric Brewer's original CAP theorem paper
- "Designing Data-Intensive Applications" - Chapter 9
- AWS DynamoDB consistency models
- Google Spanner's approach to CAP

---

**Key Takeaway**: CAP theorem giúp hiểu trade-offs khi thiết kế distributed systems. Trong thực tế, chọn CP hay AP phụ thuộc vào business requirements. 