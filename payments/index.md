# 🚀 Learning Roadmap

## 📌 Topics to Cover
- Node.js
- Redis
- RabbitMQ
- Node-RED
- LoopBack
- SQL
- Bash
- Regex
- Payments (oe-cloud)

---

## 📌 Node.js Core
- [ ] Understand event loop, async (callbacks, promises, async/await)
- [ ] Build REST API with Express
- [ ] Handle file I/O and error handling
- [ ] Unit Testing with Mocha & Chai
- [ ] **Projects**
  - [ ] REST API for User Management (CRUD)
  - [ ] File Upload/Download Service  

### 📚 Learning Resources
- [Namaste JavaScript (YouTube)](https://youtube.com/playlist?list=PLlasXeu85E9cQ32gLCvAvr9vNaUccPVNP&feature=shared)  
- [NodeJS - The Complete Guide (Udemy)](https://www.udemy.com/share/1013ho3@RYHnsYc4dbjTmctJKx444bPWcNhagY2r41_eucUlclvb_BwPHsP0JhxHmeHqtErj/)  

---

## 📌 Foundations
- [ ] **SQL DB**
  - [ ] CRUD operations, joins, indexes, transactions
  - [ ] **Project:** To-Do App with SQL DB  
        *(use Node.js + `pg` or `mysql2` driver, no ORM)*  
- [ ] **Regex**
  - [ ] Learn pattern matching, groups, lookaheads
  - [ ] **Project:** Parse log files & extract error messages  
- [ ] **Bash**
  - [ ] Learn file operations, permissions, processes  
        *(basic commands: `ls`, `lsof`, `cat`, `man`, `vim`, `nano`, `grep`, `ps`)*  
  - [ ] **Project:** Write a backup Bash script for PostgreSQL DB  

### 📚 Learning Resources
- [LinuxCommand.org – Learning the Shell](https://linuxcommand.org/lc3_learning_the_shell.php)  
- [Regexr Playground](https://regexr.com/)  

---

## 📌 Redis
- [ ] Understand Redis concepts & redis-cli
- [ ] Connect Node.js to Redis
- [ ] Use Redis for caching
- [ ] **Projects**
  - [ ] API with Redis Caching  
        *(e.g., Weather API caching last 10 queries)*  

### 📚 Learning Resources
- [Redis University – Redis for JavaScript Developers](https://university.redis.io/learningpath/jjzus2xg4rjdrt)  

---

## 📌 Messaging Systems (RabbitMQ)
- [ ] Understand exchange, queue, routing key
- [ ] Implement producer-consumer model with Node.js
- [ ] **Projects**
  - [ ] Extend Weather API project:  
        - Producer pushes weather data to queue  
        - Consumer reads from queue & stores in SQL DB  

### 📚 Learning Resources
- [RabbitMQ Tutorials (Official)](https://www.rabbitmq.com/getstarted.html)  
- [RabbitMQ in Node.js (Guide)](https://www.cloudamqp.com/blog/part1-rabbitmq-for-beginners-what-is-rabbitmq.html)  

---

## 📌 Node-RED
- [ ] Learn flows & nodes (how to create custom nodes and flows)
- [ ] **Projects**
  - [ ] Convert the Weather API + RabbitMQ project into a Node-RED flow  

### 📚 Learning Resources
- [Node-RED Documentation](https://nodered.org/docs/)  
- [Node-RED Tutorials (YouTube)](https://www.youtube.com/@Node-RED)  

---

## 📌 LoopBack 3 (LB)
- [ ] Understand LoopBack framework basics (models, datasources, repositories)
- [ ] Learn to create APIs with LoopBack CLI
- [ ] Read about Hooks, Middleware, and Remote Methods
- [ ] Integrate LoopBack with SQL DB & Redis
- [ ] **Projects**
  - [ ] Recreate the **Weather API + Redis Caching** in LoopBack  

### 📚 Learning Resources
- [LoopBack 3 Documentation](https://loopback.io/doc/en/lb3/) 

---

## 📌 Payments (oe-cloud)
- [ ] What is oe-cloud: [oe-cloud GitHub](https://github.com/EdgeVerve/oe-cloud)  
- [ ] Integrate Redis, RabbitMQ, SQL, LoopBack within payments app  
- [ ] Bring outbound app up:
    - Clone repo `oepy-outbound-payments`  
    - Checkout branch `ebrt1` *(confirm correct branch)*  
    - `yarn install`  
    - Update env variables (Redis, MQ, DB) in start file  
    - Run `node pushFlow.js` *(common flow: oepy → scheme)*  
    - Start with `sh nohup_pg_start_with_migrate_outbound.sh`  
- [ ] Understand different services involved in payments  
      *(txn-handler, reference, sanction, …)*  
- [ ] Payments process flow: NCP / RCP / RJCT / RVSL  
- [ ] PO creation process  
- [ ] CP and SP process  
- [ ] PACS messages and MsgHub  

### 📚 Learning Resources
- [oe-cloud GitHub](https://github.com/EdgeVerve/oe-cloud)  
- KT (Knowledge Transfer) sessions to be scheduled for specific modules  

---

