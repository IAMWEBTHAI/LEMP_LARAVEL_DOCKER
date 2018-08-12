# Docker LEMP+LARAVEL | nginx + php7-fpm(extentions) + mysql5.6 + phpmyadmin + laravel

Git นี้สร้างขึ้นไว้ใช้ส่วนตัวคำอธิบายต่างๆท่านอาจจะไม่เข้าใจ หากจะนำไปใช้ก็สามารถนำไปใช้ได้โดยไม่ต้องขออนุญาติใดๆ

## Getting Started

ให้ Clone โปรเจ็กจาก Git นี้ และจะได้โครงสร้างแบบนี้
```
โครงสร้างข้อมูล
MyProject
├── docker-compose.yml
├── logs [เก็บ Log ต่างๆ]
│   ├── mysql
│   ├── nginx
│   └── php-fpm
├── mysql
│   ├── backup
│   ├── backup.sh [แบ็คอัพ mysql]
│   ├── data [ข้อมูลที่เก็บใน Mysql]
│   └── initdb [sql file ที่ต้องการ import สำหรับการติดตั้งครั้งแรก]
├── nginx
│   ├── conf
│   │   └── nginx.conf [เก็บ Config ต่างๆของ Nginx1]
│   └── conf.d
│       ├── [domain].conf [เก็บ Config ของเว็บ , Root Directory, Ports, htaccess[nginx]]
│       └── phpmyadmin.conf [เก็บ Config ของ phpmyadmin , Root Directory, Ports, htaccess[nginx]]
├── php
│   ├── Dockerfile [ไฟล์ Dockerfile ติดตั้ง Extensions เพิ่มเติม + ติดตั้ง Composer]
│   ├── php-fpm.conf [ไฟล์ Config ของ php]
│   └── php.ini [Config ต่างๆของ php]
└── www
```
เสร็จแล้วให้นำโปรเจ็ค Laravel ของเราใส่ไว้ที่โฟล์เดอร์ "www" หรือ Clone โปรเจ็คไว้ที่ "www"

### Docker-compose.yml

```
version: '2'
services:
  db:
    image: mysql:5.6
    restart: always #Restart ทุกครั้งเมื่อ Service ทำงานผิดพลาด
    volumes: #คือการดึงไฟล์ใน Container มาเพื่อสามารถแก้ไขโค้ดได้ง่าย(ถ้าไฟล์ใดเปลี่ยนแปลงจะส่งผลต่อทั้งสอง Directory)
      - ./mysql/initdb/:/docker-entrypoint-initdb.d
      - ./mysql/data/:/var/lib/mysql
      - ./logs/mysql:/var/log/mysql
    environment: #setup เพิ่มเติม
      - MYSQL_ROOT_PASSWORD=รหัสผ่านRoot
      - MYSQL_DATABASE=ชื่อDATABASE
      - MYSQL_USER=ชื่อผู้ใช้
      - MYSQL_PASSWORD=รหัสผ่าน
    
  
  php:
    build: ./php #ดึงไฟล์ Dockerfile มาใช้
    restart: always #Restart ทุกครั้งเมื่อ Service ทำงานผิดพลาด
    volumes: #คือการดึงไฟล์ใน Container มาเพื่อสามารถแก้ไขโค้ดได้ง่าย(ถ้าไฟล์ใดเปลี่ยนแปลงจะส่งผลต่อทั้งสอง Directory)
      - ./www/:/var/www/html/
      - ./php/php-fpm.conf:/usr/local/etc/php-fpm.conf
      - ./php/php.ini:/usr/local/etc/php/php.ini
      - ./logs/php-fpm:/var/log/php-fpm
    expose: #เปิดพอร์ตให้ทำงานในฝั่ง Container อย่างเดียว
      - "9000"
  
  nginx:
    image: nginx:stable-alpine #ดึงไฟล์ Image มาจาก Hub.Docker
    restart: always #Restart ทุกครั้งเมื่อ Service ทำงานผิดพลาด
    volumes:
      - ./nginx/conf/nginx.conf:/etc/nginx/conf/nginx.conf:ro
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./logs/nginx:/var/log/nginx
    volumes_from:
      - php
    ports: #เปิดพอร์ตให้ทำงานในทั้งสองฝั่ง Host:Container
      - "80:80"
  

  pma:
    image: phpmyadmin/phpmyadmin #ดึงไฟล์ Image มาจาก Hub.Docker
    restart: always #Restart ทุกครั้งเมื่อ Service ทำงานผิดพลาด
    environment: #setup เพิ่มเติม
      - PMA_HOST=db #ให้ดึง Host มาจาก db container
      - PMA_ARBITRARY=1
    ports: #เปิดพอร์ตให้ทำงานในทั้งสองฝั่ง Host:Container
      - "8000:80"
```

### [domain].conf เปลี่ยนชื่อ domain เป็นชื่อเว็บเรา

```
server {
   charset utf-8;
   client_max_body_size 128M;

   listen 80; ## listen for ipv4
   #listen [::]:80 default_server ipv6only=on; ## listen for ipv6

   server_name domain.me; #Domain name
   root /var/www/html/domain/web; #Directory หลักของเว็บ
   index index.php;

   access_log  /var/log/nginx/domain-access.log; #เก็บLog
   error_log   /var/log/nginx/domain-error.log; #เก็บLog

   location / {
       # Redirect everything that isn't a real file to index.php
       try_files $uri $uri/ /index.php$is_args$args;
       # ส่วนนี้เหมือน .htaccess ของ Apache ต้องใส่ไม่งั้นจะไม่สามารถทำงานในส่วนของ Route ไม่ได้
   }

   # uncomment to avoid processing of calls to non-existing static files by Yii
   #location ~ \.(js|css|png|jpg|gif|swf|ico|pdf|mov|fla|zip|rar)$ {
   #    try_files $uri =404;
   #}
   #error_page 404 /404.html;

   location ~ \.php$ {
       include fastcgi_params;
       fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
       fastcgi_pass   php:9000; # ส่งข้อมูลเว็บไปให้ php container ผ่านพอร์ต 9000 เพื่อให้ php ประมวณผลภาษา PHP
       try_files $uri =404;
   }

   location ~ /\.(ht|svn|git) {
       deny all;
   }
}
```

## ขั้นตอนการติดตั้ง

* ให้วางโครงสร้างข้อมูล
* นำ Laravel มาไว้ใน "www"
* docker-compose up -d ติดตั้ง Container และ Run
* docker exec -it php sh เข้าใช้งาน Container
* หลังเข้า exec ให้ใช้คำสั่ง php composer install ติดตั้ง Laravel
* แก้ไขไฟล์ใน .env
* --------------- เสร็จสิ้นพร้อมใช้งาน ---------------

## Authors

[IAMTHEME จำหน่ายสคริปเว็บ](https://www.iamtheme.com/)
## License

This project is licensed under the MIT License


