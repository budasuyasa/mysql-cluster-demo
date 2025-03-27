
Dalam latihan ini kita akan mencoba untuk membuat sebuah cluster MySQL basis data. Kita akan menggunakan Docker untuk proses belajar ini.

Pertama buat sebuah directory baru untuk menempatkan file project latihan. Misalkan dengan nama mysql-cluster. Di dalam directory mysql-cluster, buat sebuah  file dengan nama docker-compose.yml. Isikan fil tersebut dengan code berikut:

```yaml
services:
  master:
    image: mysql:8
    container_name: bdl-master
    environment:
      MYSQL_ROOT_PASSWORD: telurrebus
      MYSQL_DATABASE: appku
      MYSQL_USER: bang_kopi
      MYSQL_PASSWORD: telurrebus
    command: >
      --server-id=1
      --log-bin=mysql-bin
      --binlog-do-db=test_db
      --binlog_format=ROW
    ports:
      - "3307:3306"
    networks:
      - appku-net

  slave1:
    image: mysql:8
    container_name: bdl-slave1
    environment:
      MYSQL_ROOT_PASSWORD: telurrebus
      MYSQL_DATABASE: appku
      MYSQL_USER: bang_kopi
      MYSQL_PASSWORD: telurrebus
    command: >
      --server-id=2
      --relay-log=mysql-relay-bin
      --log-bin=mysql-bin
      --read-only=ON
    ports:
      - "3308:3306"
    depends_on:
      - master
    networks:
      - appku-net

  slave2:
    image: mysql:8
    container_name: bdl-slave2
    environment:
      MYSQL_ROOT_PASSWORD: telurrebus
      MYSQL_DATABASE: appku
      MYSQL_USER: bang_kopi
      MYSQL_PASSWORD: telurrebus
    command: >
      --server-id=3
      --relay-log=mysql-relay-bin
      --log-bin=mysql-bin
      --read-only=ON
    ports:
      - "3309:3306"
    depends_on:
      - master
    networks:
      - appku-net

networks:
  appku-net:

```

Di dalam `docker-compose.yml` di atas kita akan membuat 3 buah node MySQL yang terdiri dari 1 **master** database dan **2 node** slave. Master berfungsi sebagai basis data utama, sumber data yang terkini ada di dalam master. Sedangkan slave, akan kita gunakan sebagai replikasi dan backup. Masing-masing slave hanya kita gunakan untuk membaca data, jadi perubahan data pada slave tidak diizinkan. Ini secara explisit telah kita konfigurasikan pada parameter `—read-only=ON` .

### Kenapa read only?

Metode master slave adalah metode replikasi sederhana yang bisa kita gunakan untuk mendemonstrasikan clustering basis data. Biasanya metode ini tidak akan kita gunakan pada lingkungan production, namun sudah cukup menggambarkan bagaimana clustering bekerja. 

Untuk lingkungan production kita akan menggunakan image `mysql/cluster` yang sudah production-ready untuk clustering. `mysql/cluster` menggunakan engine database yang berbeda yaitu NDB (Network Databse) yang teroptimasi untuk basis data terdistribusi.

## Setup database client dengan Datagrip (atau client lainnya)

Untuk memudahkan berinteraksi dengan node master dan slave, gunakan aplikasi client seperti Datagrip. Buat sebuah project baru dengan Datagrip kemudian, tambahkan setiap node sebagai data source.  Untuk membuat project baru bisa dengan:

1. File → New Project
2. Berikan nama project, misalkan `dbl-cluster`
3. Tambahkan data source baru ke dalam project dengan klik icon Plus (+) → MySQL → MySQL
    
    ![Screenshot 2025-03-26 172646.png](attachment:4b58b069-03a6-48b3-afb9-b651452e9e1c:Screenshot_2025-03-26_172646.png)
    
4. Gunakan user root dan masukan credentials lain sesuai dengan docker-compose.yml
    
    ![Screenshot 2025-03-26 180454.png](attachment:6d59f642-a662-49b6-8c66-4cc541fa8f29:Screenshot_2025-03-26_180454.png)
    
5. Lakukan langkah 4 untuk data source dari master, slave1, dan slave3.

### Melakukan konfigurasi pada master

Kita perlu melakukan konfigurasi pada node master untuk mengijinkan replikasi dari semua slave. 

Klik kanan panan master dan Buka dari node master → New → Query Console.

 Lakukan eksekusi query berikut:

```yaml
GRANT REPLICATION SLAVE ON *.* TO 'bang_kopi'@'%';
FLUSH PRIVILEGES;
SHOW BINARY LOG STATUS;
```

Catat value dari kolom File dan Position karena akan kita gunakan pada tahap berikutnya

 

![Screenshot 2025-03-26 183047.png](attachment:1fa48f16-a06c-4ec9-866f-f0c2be1d0a16:Screenshot_2025-03-26_183047.png)

### Melakukan konfigurasi pada slave

Ekskusi query berikut pada node slave:

```yaml
STOP REPLICA;

CHANGE REPLICATION SOURCE TO 
    SOURCE_HOST='master',
    SOURCE_USER='bang_kopi',
    SOURCE_PASSWORD='telurrebus',
    SOURCE_LOG_FILE='mysql-bin.000004',
    SOURCE_LOG_POS=582;

START REPLICA;

SHOW REPLICA STATUS;
```

`SHOW REPLICA STATUS` akan menampilkan apakah replica sudah terhubung. Lakukan tahap di atas pada semua node slave (slave1 dan slave2).

### Test replikasi

Kita bisa melakukan uji coba replikasi dengan membuat table baru pada database `appku` pada node master. Setelah itu, cek node slave, dan pastikan table baru tersebut juga muncul. 

Selamat mencoba!

### Download project:

Contoh project dapat didownload via Github repository berikut:
https://github.com/budasuyasa/mysql-cluster-demo

