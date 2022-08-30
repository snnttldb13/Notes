# Dataguard kurulumu için yapılacak işlemler aşağıda anlatılmıştır

### Standby Server Hazırlanması

- Standby sunucuda işletim sistemi ve oracle yazılımı kurulduktan sonra ilk adım olarak primary ve standby sunucuların IP ve hostname bilgileri hosts dosyasına yazılır.
```
nano /etc/hosts

10.0.0.1    orcl
10.0.0.2    sby1
```

- Network ayarlarının devamında tnsnames için $ORACLE_HOME/network/admin dizinine tnsnames.ora dosyası oluşturulur ve içine  tns bilgileri girilir, ardından listener başlatılır.

```
nano $ORACLE_HOME/network/admin/tnsnames.ora
```
```

ORCL =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = lnxdbsrv )(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = orcl.karmed)
    )
  )
sby1 =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = sby1)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SID = sby1)
    )
  )
 
 
 lsnrctl start
```

  
 - Standby sunucusu için oracle başlangıç parametrelerini belirliyoruz. Bunun için arzu ettiğimiz bir dizinde pfile.ora isimli bir dosya oluşturyor ve içine gerekli parametreleri ekliyoruz. 
        
 ```
    nano /home/oracle/pfile.ora
 ```
 ```
    Db_name=orcl
    Db_unique_name=sby1
    Compatible=11.2.0.4.0
    Db_file_name_convert='/orcl/','/sby1/'
    db_recovery_file_dest_size=100G
    db_recovery_file_dest='/u02/fra'
    *.sga_max_size=30G
    *.sga_target=30G
    *.open_cursors=750
    *.processes=1500
```
 
- Oluşturuan parametre dosyası ile standby sunucuda oracle'ı başlatıyor ve parametre dosyasından spfile oluşturuyoruz.

```
SQL> startup nomount pfile='/c_4/pfile.ora' ;
SQL> create spfile from pfile='/c_4/pfile.ora' ; 
SQL> shu immediate ; 
SQL> startup nomount;
``` 

- Database broker'ı ayarlıyoruz.

```
SQL> alter system set dg_broker_Start=true;
```

- Replikasyon kurulumu için yedek alırken network paylaşımlı bir disk kullanabiliriz. NFS kullanmak için diskin paylaşılacağı sunucuda yedekleme için bir dizin oluşturuyoruz. 

```
mkdir  /backup
chown oracle:oinstall /backup
``` 

- Bu dizini NFS servisine gösteriyoruz. 

``` 
nano /etc/exports
```
```
/backup               *(rw,sync,no_wdelay,insecure_locks,no_root_squash)
``` 

- Ardından NFS servisini başlatıyoruz.

``` 
service nfs restart
``` 
 
### Primary Server Hazırlanması

- Önce hosts dosyası güncellenir. 
 
```
nano /etc/hosts
```
```
10.0.0.1    orcl
10.0.0.2    sby1
```
- Ardından , tnsnames dosyası güncellenir.

```
nano $ORACLE_HOME/network/admin/tnsnames.ora
```
```
ORCL =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = lnxdbsrv )(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = orcl.karmed)
    )
  )
sby1 =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = sby1)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SID = sby1)
    )
  )
```

- Eğer disk standby sunucudan paylaşıldı ise diski primary sunucuya mount ediyoruz.

```
nano /etc/fstab
```
```
sby1:/backup /backup  nfs rw,bg,hard,nointr,tcp,vers=3,timeo=300,rsize=32768,wsize=32768,actimeo=0       0 0
```
- Eğer yoksa primary sunucuda yedekleme için standby sunucuda açılan klasörle aynı isimde bir klasör açıyoruz.

```
mkdir /backup
mount /backup
```		

- Eğer primary sunucu arşiv modda değilse arşiv moda alıyor ve loglamayı zorunlu kılıyoruz. Arşiv mod durumunu öğrenmek ve gerekli ayarları yapmak için;

```
SQL> select log_mode from v$database;

LOG_MODE
------------
NOARCHIVELOG
```
```
SQL> shu immediate;
SQL> startup mount;
SQL> alter database archivelog;
SQL> alter database open;
SQL> alter database force logging ;
```

- Primary sunucunun redolog boyutunu ve sayısını kontrol ediyoruz. Eğer zaten eklenmemiş ise aynı boyut ve sayıda standby redolog ekliyoruz.

```
SQL> select group#,thread#,bytes from v$log;
        GROUP#    THREAD#      BYTES
---------- ---------- ----------
         1          1  209715200
         2          1  209715200
         3          1  209715200

SQL>  select group#,thread#,bytes from v$standby_log;
satir secilmedi

SQL> alter database add standby logfile size 200M;
SQL> alter database add standby logfile size 200M;
SQL> alter database add standby logfile size 200M;
```

- Database broker'ı ayarlıyoruz.

```
SQL> alter system set dg_broker_Start=true;
```

- Log transferi için gerekli ayarları kontrol ediyor ve ayarlamaları yapıyoruz. İlgili parametreler daha önceden ayarlanmış olabilir, farklı standby sunucular için kullanılıyor olabilir. Bu sebeple değerleri ayarlamadan önce kontrol etmek önemli. 

```
SQL> show parameter log_archive_config;
NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
log_archive_config                   string

SQL>  show parameter  log_Archive_dest_2;

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
log_archive_dest_2                   string



SQL> alter system set log_archive_config='dg_config=(orcl,sby1)';
SQL> alter system set log_archive_dest_2='SERVICE=sby1 async valid_for=(online_logfile,primary_role) db_unique_name=sby1';
```

-  Parola dosyasını standby sunucuya kopyalıyoruz.

``` 
 scp $ORACLE_HOME/dbs/orapworcl sby1:$ORACLE_HOME/dbs/orapwsby1
```

- Yedek almak için gerekli script dosyasını oluşturyor ve içine gerekli parametreleri ekliyoruz.
```
nano /home/oracle/yedekal.rman
```
```
run {
    allocate channel C1 type disk;
    allocate channel C2 type disk;
	allocate channel C3 type disk;
    BACKUP as compressed backupset  incremental level 0 database format '/backup/DbBck_%T_%U.karmed' PLUS ARCHIVELOG;
    BACKUP FORMAT '/backup/CtrlFile%U' CURRENT CONTROLFILE FOR STANDBY;
    release channel C1;
    release channel C2;
	release channel C3;
}
```


- Bütün hazırlıklar tamam. Şimdi sıra yedek almada. Yedek alma işini sunucunun kapasitesine göre paralel yapabiliriz. Daha fazla işlemci tüketecek ancak daha hızlı olacaktır. Yedekleme işleminin kesintiye uğramaması için screen penceresi içersinde başlatıyoruz.

```
screen
cd 
rman target / log=/home/oracle/dgicinyedek.log
RMAN> @yedekal.rman
RMAN> exit
```
		
### Duplicate işlemleri

- Standby sunucuya oracle kullancısı ile login oluyoruz. Ardından RMAN ile hem primary hem de standby sunucuya bağlanıyor ve duplicate komutu veriyoruz. İşlemlerin kesintiye uğramaması için screen kullanıyoruz, ayrıca -L parametresi ile de screen ile log tutuyoruz, olası hatada ne olduğunu anlamak için kullanabiliriz.
 
```
screen -L
rman target sys@orcl auxiliary /
RMAN> DUPLICATE TARGET DATABASE FOR STANDBY NOFILENAMECHECK;
RMAN> exit
```	

- Duplicate işlemi tamamlandıktan sonra standby sunucuda standby redolog dosyalarını kontrol ediyoruz. Normal şartlarda duplicate işlemi ile birlikte standby redolog dosyaları da otomatik olarak oluşuyor ancak bazı durumlarda hatalar olabiliyor. Kontrol etmek bu açıdan önemli. Standby redologların sayısı ve boyutu replikasyonun sağlıklı çalışabilmesi için yüksek öneme sahip.

```	
SQL>  select group#,thread#,bytes from v$standby_log;
```

- Eşitlemenin sağlanabilmesi için recovery process'ini başlatıyoruz.

```	
SQL> alter database recover managed standby database using current logfile disconnect;
```

### Dataguard Broker yapılandırması

- Primary sunucuya oracle ile login oluyor ve dgmgrl aracı ile konfigürasyon oluşturuyoruz.

```	
 dgmgrl /
 DGMGRL> create configuration 'DGConfig1' as   primary database is 'orcl' connect identifier is 'orcl';
 DGMGRL> add database sby1 as connect identifier is sby1;
 DGMGRL> enable configuration;
 DGMGRL> exit
```	

- Kurulum işlemleri artık tamamlandı. Konfigürasyonun ve standby sunucunun durumu görmek için dgmgrl aracı kullanılabilir.
```	
 DGMGRL> show configuration ;
 Configuration - dgconfig1
  Protection Mode: MaxPerformance
  Databases:
    orcl - Primary database
    sby1 - Physical standby database
 Fast-Start Failover: DISABLED
 Configuration Status:
 SUCCESS
 DGMGRL> 
 DGMGRL> 
 DGMGRL> show database sby1 ;
 Database - sby1
  Role:            PHYSICAL STANDBY
  Intended State:  APPLY-ON
  Transport Lag:   0 seconds
  Apply Lag:       0 seconds
  Real Time Query: OFF
  Instance(s):
    sby1
 Database Status:
 SUCCESS
 DGMGRL> exit
```
